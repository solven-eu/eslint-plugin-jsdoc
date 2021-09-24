<div id="app">
  {{ message }}
</div>
<script>

var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  },
  methods: {
    /**
     * @description jsLint throws errors
     * @see https://someWebSite.com/
     * @param {number} param - The number to be added
     * @return {number} 2 + param
     */
    // eslint-disable-next-line no-unused-vars
    twoplusparam(param) { return 2 + param; }
  }
});

</script>
