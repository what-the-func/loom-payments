<template lang="html">
  <v-container
    v-if="ready"
    grid-list-md
    text-xs-center
  >
    <v-layout
      row
      pb-5
    >
      <v-flex xs6>
        <v-card>
          <v-card-title>
            <span class="display-1">Rinkeby Balance</span>
            <span class="subheading">{{ rinkebyWalletAddr }}</span>
          </v-card-title>
          <v-card-text class="display-2">{{ rinkebyBalance }}</v-card-text>
        </v-card>
      </v-flex>
      <v-flex xs6>
        <v-card>
          <v-card-title>
            <span class="display-1">Loom Balance</span>
            <span class="subheading">{{ loomWalletAddr }}</span>
          </v-card-title>
          <v-card-text class="display-2">{{ loomBalance }}</v-card-text>
        </v-card>
      </v-flex>
    </v-layout>
    <v-layout
      row
      pb-5
    >
      <v-flex xs12>
        <div class="display-1">
          Make a payment of 10 FUNC<v-btn
            :disabled="busy"
            color="success"
            large
            @click="pay"
          >Pay Now</v-btn>
        </div>
      </v-flex>
    </v-layout>
    <v-layout row>
      <v-flex xs6>
        <v-btn
          :disabled="busy"
          @click="faucet"
        >
          Get More FUNC
        </v-btn>
        <v-btn
          :disabled="busy"
          @click="deposit"
        >
          Deposit To Loom
        </v-btn>
      </v-flex>
      <v-flex xs6>
        <v-btn
          :disabled="busy"
          @click="withdraw"
        >
          Withdraw From Loom
        </v-btn>
      </v-flex>
    </v-layout>
  </v-container>
</template>

<script>
import Web3 from 'web3'
import {
  Address,
  Client,
  Contracts,
  CryptoUtils,
  EthersSigner,
  LocalAddress,
  LoomProvider,
  NonceTxMiddleware,
  SignedTxMiddleware
} from 'loom-js'
import { ethers } from 'ethers'
import BN from 'bn.js'
import { truncate } from 'fs'

const RINKEBY_TOKEN_ADDR = '0xEe3cb16e758dF2bE1a95629581cAb9c1CD77BB0D'
const LOOM_TOKEN_ADDR = '0xF947938cc91dB9E306941Fb91e8C33E5722EEd8C'
const PAYMENT_ADDR = '0xeb8372e83ABA83715500DAb799EBb882973FfeCC'
const TRANSFER_GATEWAY_ADDR = '0xb73C9506cb7f4139A4D6Ac81DF1e5b6756Fab7A2'

export default {
  data() {
    return {
      ready: false,
      rinkebyProvider: null,
      rinkebyToken: null,
      rinkebyGateway: null,
      rinkebyWalletAddr: null,
      rinkebyBalance: 0,
      loomClient: null,
      loomProvider: null,
      loomToken: null,
      loomGateway: null,
      loomWalletAddr: null,
      loomBalance: 0,
      busy: false
    }
  },

  async mounted() {
    await this.initWeb3()
    await this.initLoom()
    await this.initContracts()
    await this.mapAddresses()
    await this.addEventListeners()
    await this.updateBalances()
  },

  methods: {
    async initWeb3() {
      let web3js
      if (window.ethereum) {
        window.web3 = new Web3(ethereum)
        web3js = new Web3(ethereum)
        await ethereum.enable()
      } else if (window.web3) {
        window.web3 = new Web3(window.web3.currentProvider)
        web3js = new Web3(window.web3.currentProvider)
      } else {
        alert('Metamask is not Enabled')
      }

      if (web3js) {
        this.web3js = web3js
        this.rinkebyProvider = new ethers.providers.Web3Provider(
          web3js.currentProvider
        )

        this.rinkebyWalletAddr = (await this.rinkebyProvider.listAccounts())[0]
      }
    },

    async initLoom() {
      const client = new Client(
        'extdev-plasma-us1',
        'wss://extdev-plasma-us1.dappchains.com/websocket',
        'wss://extdev-plasma-us1.dappchains.com/queryws'
      )
      this.loomClient = client

      const storedKey = localStorage.getItem('loomKey')

      let privKey
      if (storedKey) {
        privKey = CryptoUtils.B64ToUint8Array(storedKey)
      } else {
        privKey = CryptoUtils.generatePrivateKey()
        localStorage.setItem('loomKey', CryptoUtils.Uint8ArrayToB64(privKey))
      }
      const pubKey = CryptoUtils.publicKeyFromPrivateKey(privKey)
      this.loomWalletAddr = LocalAddress.fromPublicKey(pubKey).toString()

      // required middleware
      client.txMiddleware = [
        new NonceTxMiddleware(pubKey, client),
        new SignedTxMiddleware(privKey)
      ]

      const provider = new LoomProvider(client, privKey)

      const web3Provider = new Web3(provider)
      this.loomProvider = new ethers.providers.Web3Provider(
        web3Provider.currentProvider
      )
    },

    async mapAddresses() {
      const rinkebyAddress = Address.fromString(`eth:${this.rinkebyWalletAddr}`)
      const loomAddress = Address.fromString(
        `${this.loomClient.chainId}:${this.loomWalletAddr}`
      )
      const addressMapper = await Contracts.AddressMapper.createAsync(
        this.loomClient,
        loomAddress
      )
      const hasMapping = await addressMapper.hasMappingAsync(loomAddress)
      if (hasMapping) {
        const mapping = await addressMapper.getMappingAsync(loomAddress)
        if (mapping.to.toString() !== rinkebyAddress.toString()) {
          alert(
            `Loom account is linked to a different account please switch to: ${mapping.to.local.toString()}`
          )
          return
        }
      } else {
        alert(
          'Please map your Ethereum and Loom accounts by signing the next prompt'
        )
        await addressMapper.addIdentityMappingAsync(
          loomAddress,
          rinkebyAddress,
          new EthersSigner(this.rinkebyProvider.getSigner())
        )
      }
      this.ready = true
    },

    async initContracts() {
      const tokenABI = [
        'function faucet() public @30000',
        'function balanceOf(address who) external view returns (uint256)',
        'function approve(address spender, uint256 value) external returns (bool) @50000',
        'function transfer(address to, uint256 value) external returns (bool) @30000',
        'event Transfer(address indexed from, address indexed to, uint256 value)'
      ]

      const gatewayABI = [
        'function depositERC20(uint256 amount, address contractAddress) external @50000',
        'function withdrawERC20(uint256 amount, bytes sig, address contractAddress) external @60000'
      ]

      const payABI = ['function pay(uint256 amount) external @50000']

      this.rinkebyToken = new ethers.Contract(
        RINKEBY_TOKEN_ADDR,
        tokenABI,
        this.rinkebyProvider.getSigner()
      )

      this.loomToken = new ethers.Contract(
        LOOM_TOKEN_ADDR,
        tokenABI,
        this.loomProvider.getSigner()
      )

      this.payContract = new ethers.Contract(
        PAYMENT_ADDR,
        payABI,
        this.loomProvider.getSigner()
      )

      this.rinkebyGateway = new ethers.Contract(
        TRANSFER_GATEWAY_ADDR,
        gatewayABI,
        this.rinkebyProvider.getSigner()
      )

      const loomAddress = Address.fromString(
        `${this.loomClient.chainId}:${this.loomWalletAddr}`
      )
      this.loomGateway = await Contracts.TransferGateway.createAsync(
        this.loomClient,
        loomAddress
      )
    },

    async addEventListeners() {
      let rinkebyReceiveFilter = this.rinkebyToken.filters.Transfer(
        null,
        this.rinkebyWalletAddr
      )
      this.rinkebyToken.on(rinkebyReceiveFilter, (from, to, value) => {
        this.updateBalances()
      })

      let rinkebySendFilter = this.rinkebyToken.filters.Transfer(
        this.rinkebyWalletAddr,
        null
      )
      this.rinkebyToken.on(rinkebySendFilter, (from, to, value) => {
        this.updateBalances()
      })

      let loomReceiveFilter = this.loomToken.filters.Transfer(
        null,
        this.loomWalletAddr
      )
      this.loomToken.on(loomReceiveFilter, (from, to, value) => {
        this.updateBalances()
      })
      let loomSendFilter = this.loomToken.filters.Transfer(
        this.loomWalletAddr,
        null
      )
      this.loomToken.on(loomSendFilter, (from, to, value) => {
        this.updateBalances()
      })
    },

    async updateBalances() {
      const rinkebyBalance = await this.rinkebyToken.balanceOf(
        this.rinkebyWalletAddr,
        {
          from: this.rinkebyWalletAddr
        }
      )
      this.rinkebyBalance = ethers.utils.formatEther(rinkebyBalance.toString())

      const loomBalance = await this.loomToken.balanceOf(this.loomWalletAddr, {
        from: this.loomWalletAddr
      })
      this.loomBalance = ethers.utils.formatEther(loomBalance.toString())
    },

    async deposit() {
      this.busy = true

      const weiAmount = ethers.utils.parseEther('50')

      await this.rinkebyToken.approve(TRANSFER_GATEWAY_ADDR, weiAmount)
      await this.rinkebyGateway.depositERC20(weiAmount, RINKEBY_TOKEN_ADDR)

      this.busy = false
    },

    async withdraw() {
      this.busy = true

      this.loomGateway.once(
        Contracts.TransferGateway.EVENT_TOKEN_WITHDRAWAL,
        async event => {
          const tokenEthAddr = Address.fromString(`eth:${RINKEBY_TOKEN_ADDR}`)
          const ownerRinkebyAddr = Address.fromString(
            `eth:${this.rinkebyWalletAddr}`
          )

          if (
            event.tokenContract.toString() === tokenEthAddr.toString() &&
            event.tokenOwner.toString() === ownerRinkebyAddr.toString()
          ) {
            await this.rinkebyGateway.withdrawERC20(
              ethers.utils.parseEther(this.loomBalance).toString(),
              event.sig,
              RINKEBY_TOKEN_ADDR
            )
            this.busy = false
          }
        }
      )

      const gwAddress = this.loomGateway.address.local.toString()
      const tokenAddress = Address.fromString(
        `${this.loomClient.chainId}:${LOOM_TOKEN_ADDR}`
      )
      const weiAmount = ethers.utils.parseEther(this.loomBalance).toString()

      await this.loomToken.approve(gwAddress, weiAmount)
      await this.loomGateway.withdrawERC20Async(new BN(weiAmount), tokenAddress)
    },

    async pay() {
      this.busy = true
      const weiAmount = ethers.utils.parseEther('10').toString()

      await this.loomToken.functions.approve(PAYMENT_ADDR, weiAmount)
      await this.payContract.pay(weiAmount)
      this.busy = false
    },

    async faucet() {
      this.busy = true
      await this.rinkebyToken.faucet()
      this.busy = false
    }
  }
}
</script>
