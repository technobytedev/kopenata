<template>
  <div class="login-page">
    <div class="login-card">
      <div class="coffee-icon">☕</div>
      <h1>KOPENATA</h1>
      <p class="tagline">just a scheduled mid-coffee hour<br/>with peers</p>
      <button class="google-btn" @click="signIn">
        <svg width="18" height="18" viewBox="0 0 18 18"><path fill="#4285F4" d="M17.64 9.2c0-.637-.057-1.251-.164-1.84H9v3.481h4.844c-.209 1.125-.843 2.078-1.796 2.717v2.258h2.908c1.702-1.567 2.684-3.874 2.684-6.615z"/><path fill="#34A853" d="M9 18c2.43 0 4.467-.806 5.956-2.184l-2.908-2.258c-.806.54-1.837.86-3.048.86-2.344 0-4.328-1.584-5.036-3.711H.957v2.332C2.438 15.983 5.482 18 9 18z"/><path fill="#FBBC05" d="M3.964 10.707c-.18-.54-.282-1.117-.282-1.707s.102-1.167.282-1.707V4.961H.957C.347 6.175 0 7.55 0 9s.348 2.825.957 4.039l3.007-2.332z"/><path fill="#EA4335" d="M9 3.58c1.321 0 2.508.454 3.44 1.345l2.582-2.58C13.463.891 11.426 0 9 0 5.482 0 2.438 2.017.957 4.961L3.964 7.293C4.672 5.166 6.656 3.58 9 3.58z"/></svg>
        Sign in with Google
      </button>
    </div>
  </div>
</template>

<script setup>
const supabase = useSupabaseClient()

async function signIn() {
  await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: { redirectTo: `${window.location.origin}/confirm` }
  })
}
</script>

<style scoped>
.login-page {
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--cream);
  background-image: 
    radial-gradient(circle at 20% 50%, rgba(123,79,46,0.08) 0%, transparent 50%),
    radial-gradient(circle at 80% 20%, rgba(201,168,124,0.15) 0%, transparent 40%);
}
.login-card {
  text-align: center;
  padding: 3rem 4rem;
  background: var(--card-bg);
  border: 2px solid var(--coffee);
  box-shadow: 8px 8px 0 var(--coffee);
}
.coffee-icon { font-size: 4rem; margin-bottom: 1rem; animation: steam 2s ease-in-out infinite; }
@keyframes steam { 0%,100% { transform: translateY(0); } 50% { transform: translateY(-8px); } }
h1 { font-family: "Berkshire Swash", serif; font-size: 3rem; color: var(--brown); letter-spacing: 0.1em; }
.tagline { margin: 0.75rem 0 2rem; color: var(--coffee); font-style: italic; line-height: 1.6; }
.google-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 2rem;
  background: white;
  border: 2px solid var(--brown);
  cursor: pointer;
  font-family: "Courier Prime", monospace;
  font-size: 1rem;
  font-weight: 700;
  color: var(--brown);
  transition: all 0.15s;
  box-shadow: 4px 4px 0 var(--brown);
}
.google-btn:hover { transform: translate(-2px, -2px); box-shadow: 6px 6px 0 var(--brown); }
.google-btn:active { transform: translate(2px, 2px); box-shadow: 2px 2px 0 var(--brown); }
</style>