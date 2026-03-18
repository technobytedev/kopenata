<template><div style="display:grid;place-items:center;height:100vh">☕ Brewing...</div></template>
<script setup>
// Supabase module handles the OAuth callback automatically
// After session is set, redirect home
const user = useSupabaseUser()
watchEffect(() => { if (user.value) navigateTo('/') })
</script>