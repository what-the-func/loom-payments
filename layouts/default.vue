<template>
  <v-app id="inspire">
    <v-toolbar
      color="indigo"
      dark
      fixed
      app
    >
      <v-toolbar-title>Loom Payments</v-toolbar-title>
    </v-toolbar>
    <v-content>
      <v-container
        fluid
        fill-height
      >
        <v-layout
          justify-center
          align-center
        >
          <v-flex text-xs-center>
            <nuxt/>
          </v-flex>
        </v-layout>
      </v-container>
    </v-content>
    <v-footer
      color="indigo"
      app
    >
      <span class="white--text">What The Func &copy; 2019</span>
    </v-footer>
  </v-app>
</template>

<script>
export default {
  props: {
    source: {
      type: String,
      default: ''
    }
  },
  data: () => ({
    drawer: null
  }),
  methods: {
    foo() {}
  }
}
</script>
