<template>
<div :class="['cell', { 'active': cell.active }, { 'silent': cell.active && cell.silent }]" @click="toggle()">
  <span v-if="cell.voiced" :class="[{ 'c': cell.isC }]">{{ cell.note }}</span>
</div>
</template>
<script>
export default {
  name: 'Container',
  props: [ 'cell' ],
  methods: {
    toggle () {
      this.$emit('toggle')
    }
  }
}
</script>
<style scoped>
.cell {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 3rem;
  height: 3rem;
  border-left: 1px solid #303;
  border-bottom: 1px solid #303;
  border-radius: 1px; 
  background: ghostwhite;
  cursor: pointer;
  color: grey;
  font-size: 0.8em;
}
.active {
  background: #303;
  color: ghostwhite;
}
.c {
  font-weight: bold;
}
.silent {
  background: grey;
  color: ghostwhite;
}
</style>
