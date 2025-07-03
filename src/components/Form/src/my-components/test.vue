<template>
    <input type = "text" :value="value" @input="onInput">
</template>

<script lang="ts">
import { defineComponent } from 'vue';

export default defineComponent({
    name:"MyInput",
    props:{
        value:String,
    },
    emits:['input'];
    setup(_props,{emit}){
        function onInput(e){
            emit('input',e.target.value)
        }
        return {onInput}
    },
})
</script>