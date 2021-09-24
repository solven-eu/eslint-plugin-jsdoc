<template>
  <div id="myApplication">
    <h1>Hello World</h1>
    <p>little text behind the title</p>
  </div>
</template>

<script>
export default {
  name: "app",
  methods: {
    /**
     * @description jsLint throws errors
     * @see https://someWebSite.com/
     * @param {int} param - The number to be added
     * @returns {int} 2 + param
     */
    // eslint-disable-next-line no-unused-vars
    twoplusparam(param) {
      return 2 + param;
    },
  },
};
</script>

<style>
#app {
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>