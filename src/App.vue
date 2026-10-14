<script setup>
import { computed, onMounted, ref } from 'vue'
import { createClient } from '@supabase/supabase-js'
import {
  IonApp,
  IonButton,
  IonContent,
  IonHeader,
  IonIcon,
  IonInput,
  IonItem,
  IonLabel,
  IonPage,
  IonSpinner,
  IonTitle,
  IonToolbar,
} from '@ionic/vue'
import { cameraOutline, logInOutline, personCircleOutline, saveOutline } from 'ionicons/icons'

const rawSupabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseUrl = rawSupabaseUrl?.replace(/\/(?:rest\/v1|auth\/v1|storage\/v1)\/?$/, '')
const supabaseKey = import.meta.env.VITE_SUPABASE_PUBLISHABLE_KEY || import.meta.env.VITE_SUPABASE_ANON_KEY
const connected = Boolean(supabaseUrl && supabaseKey && !supabaseUrl.includes('seu-projeto'))
const supabase = connected ? createClient(supabaseUrl, supabaseKey) : null

const view = ref('login')
const loading = ref(false)
const message = ref('')
const error = ref('')
const user = ref(null)
const form = ref({ name: '', email: '', password: '', confirmPassword: '' })
const profile = ref({ name: '', birth_date: '', nickname: '', avatar_url: '' })

const title = computed(() => ({ login: 'Entrar', signup: 'Criar conta', recovery: 'Recuperar senha', welcome: 'Olá!', profile: 'Meu perfil' }[view.value]))

function clearFeedback() {
  message.value = ''
  error.value = ''
}

async function login() {
  clearFeedback()
  if (!form.value.email || !form.value.password) {
    error.value = 'Preencha seu e-mail e sua senha.'
    return
  }
  loading.value = true
  if (supabase) {
    const { data, error: loginError } = await supabase.auth.signInWithPassword(form.value)
    if (loginError) error.value = loginError.message
    else await enterApp(data.user)
  } else {
    await enterApp({ email: form.value.email })
  }
  loading.value = false
}

async function signup() {
  clearFeedback()
  if (!form.value.name || !form.value.email || !form.value.password || !form.value.confirmPassword) {
    error.value = 'Preencha todos os campos.'
    return
  }
  if (form.value.password.length < 6) {
    error.value = 'A senha precisa ter pelo menos 6 caracteres.'
    return
  }
  if (form.value.password !== form.value.confirmPassword) {
    error.value = 'As senhas não são iguais.'
    return
  }
  loading.value = true
  if (supabase) {
    const { data, error: signupError } = await supabase.auth.signUp({
      email: form.value.email,
      password: form.value.password,
      options: { data: { name: form.value.name } },
    })
    if (signupError) error.value = signupError.message
    else if (data.user && !data.session) message.value = 'Conta criada! Confira seu e-mail para confirmar o cadastro.'
    else if (data.user) await enterApp(data.user)
  } else {
    await enterApp({ email: form.value.email, user_metadata: { name: form.value.name } })
  }
  loading.value = false
}

async function enterApp(currentUser) {
  user.value = currentUser
  profile.value = { ...profile.value, name: currentUser?.user_metadata?.name || '' }
  if (supabase && currentUser) {
    const { data } = await supabase.from('profiles').select('*').eq('id', currentUser.id).maybeSingle()
    if (data) profile.value = { ...profile.value, ...data }
  }
  view.value = 'welcome'
}

async function recoverPassword() {
  clearFeedback()
  if (!form.value.email) {
    error.value = 'Digite seu e-mail para receber o link.'
    return
  }
  loading.value = true
  if (supabase) {
    const { error: resetError } = await supabase.auth.resetPasswordForEmail(form.value.email)
    if (resetError) error.value = resetError.message
    else message.value = 'Confira sua caixa de entrada para redefinir a senha.'
  } else {
    message.value = 'Modo demonstração: o link seria enviado para seu e-mail.'
  }
  loading.value = false
}

async function saveProfile() {
  clearFeedback()
  loading.value = true
  if (supabase && user.value) {
    const { error: saveError } = await supabase.from('profiles').upsert({
      id: user.value.id,
      name: profile.value.name,
      birth_date: profile.value.birth_date || null,
      nickname: profile.value.nickname,
      avatar_url: profile.value.avatar_url,
    })
    if (saveError) error.value = saveError.message
    else message.value = 'Perfil salvo com sucesso.'
  } else {
    message.value = 'Perfil salvo no modo demonstração.'
  }
  loading.value = false
}

async function uploadAvatar(event) {
  const file = event.target.files?.[0]
  if (!file) return
  if (!supabase || !user.value) {
    profile.value.avatar_url = URL.createObjectURL(file)
    message.value = 'Foto carregada no modo demonstração.'
    return
  }
  loading.value = true
  const path = `${user.value.id}/${Date.now()}-${file.name}`
  const { error: uploadError } = await supabase.storage.from('avatars').upload(path, file, { upsert: true })
  if (uploadError) error.value = uploadError.message
  else {
    const { data } = supabase.storage.from('avatars').getPublicUrl(path)
    profile.value.avatar_url = data.publicUrl
    message.value = 'Foto carregada. Clique em salvar perfil.'
  }
  loading.value = false
}

async function logout() {
  if (supabase) await supabase.auth.signOut()
  user.value = null
  view.value = 'login'
  form.value.password = ''
  clearFeedback()
}

onMounted(async () => {
  if (!supabase) return
  const { data } = await supabase.auth.getSession()
  if (data.session) await enterApp(data.session.user)
  supabase.auth.onAuthStateChange((_event, session) => {
    if (session) user.value = session.user
  })
})
</script>

<template>
  <ion-app>
    <ion-page>
      <ion-header class="topbar" v-if="view === 'welcome' || view === 'profile'">
        <ion-toolbar>
          <ion-title>Meu Perfil</ion-title>
          <ion-button slot="end" fill="clear" @click="logout">Sair</ion-button>
        </ion-toolbar>
      </ion-header>

      <ion-content :fullscreen="true">
        <main class="page-wrap" :class="{ 'page-wrap-wide': view === 'welcome' || view === 'profile' }">
          <section v-if="view === 'login' || view === 'signup' || view === 'recovery'" class="auth-card">
            <div class="brand-mark"><ion-icon :icon="personCircleOutline" /></div>
            <h1>{{ title }}</h1>
            <p class="subtitle">Acesse sua conta para continuar.</p>

            <form @submit.prevent="view === 'login' ? login() : view === 'signup' ? signup() : recoverPassword()">
              <ion-item v-if="view === 'signup'" lines="none">
                <ion-label position="stacked">Nome</ion-label>
                <ion-input v-model="form.name" placeholder="Digite seu nome" />
              </ion-item>
              <ion-item lines="none">
                <ion-label position="stacked">E-mail</ion-label>
                <ion-input v-model="form.email" type="email" placeholder="seu@email.com" />
              </ion-item>
              <ion-item v-if="view === 'login' || view === 'signup'" lines="none">
                <ion-label position="stacked">Senha</ion-label>
                <ion-input v-model="form.password" type="password" placeholder="Digite sua senha" />
              </ion-item>
              <ion-item v-if="view === 'signup'" lines="none">
                <ion-label position="stacked">Confirmar senha</ion-label>
                <ion-input v-model="form.confirmPassword" type="password" placeholder="Digite a senha novamente" />
              </ion-item>
              <ion-button expand="block" class="main-button" type="submit" :disabled="loading">
                <ion-spinner v-if="loading" name="crescent" />
                <span v-else>{{ view === 'login' ? 'Entrar' : view === 'signup' ? 'Criar conta' : 'Enviar link' }}</span>
              </ion-button>
            </form>

            <button v-if="view === 'login'" class="text-button" @click="view = 'recovery'; clearFeedback()">Esqueci minha senha</button>
            <button v-if="view === 'login'" class="text-button" @click="view = 'signup'; clearFeedback()">Ainda não tenho uma conta</button>
            <button v-if="view === 'signup' || view === 'recovery'" class="text-button" @click="view = 'login'; clearFeedback()">Voltar para o login</button>
            <p v-if="!connected" class="demo-note">Modo demonstração ativo. Configure o Supabase no arquivo .env para usar os dados reais.</p>
          </section>

          <section v-else-if="view === 'welcome'" class="welcome-card">
            <div class="welcome-icon"><ion-icon :icon="logInOutline" /></div>
            <p class="eyebrow">Área do usuário</p>
            <h1>Bem-vindo<span v-if="profile.name">, {{ profile.name }}</span>!</h1>
            <p>Você entrou na sua conta. Aproveite para completar seus dados pessoais.</p>
            <ion-button class="main-button" @click="view = 'profile'">Editar meu perfil</ion-button>
          </section>

          <section v-else class="profile-card">
            <div class="profile-heading">
              <div class="avatar" :style="profile.avatar_url ? { backgroundImage: `url(${profile.avatar_url})` } : {}">
                <ion-icon v-if="!profile.avatar_url" :icon="personCircleOutline" />
              </div>
              <div>
                <h1>Dados do perfil</h1>
                <p>{{ user?.email }}</p>
              </div>
              <label class="photo-button" title="Escolher foto"><ion-icon :icon="cameraOutline" /><input type="file" accept="image/*" @change="uploadAvatar" /></label>
            </div>
            <form @submit.prevent="saveProfile">
              <ion-item lines="none"><ion-label position="stacked">Nome completo</ion-label><ion-input v-model="profile.name" placeholder="Digite seu nome" /></ion-item>
              <ion-item lines="none"><ion-label position="stacked">Data de nascimento</ion-label><ion-input v-model="profile.birth_date" type="date" /></ion-item>
              <ion-item lines="none"><ion-label position="stacked">Nickname</ion-label><ion-input v-model="profile.nickname" placeholder="Como quer ser chamado?" /></ion-item>
              <ion-button expand="block" class="main-button" type="submit" :disabled="loading"><ion-icon slot="start" :icon="saveOutline" />Salvar perfil</ion-button>
            </form>
          </section>
          <p v-if="message" class="feedback success">{{ message }}</p>
          <p v-if="error" class="feedback error">{{ error }}</p>
        </main>
      </ion-content>
    </ion-page>
  </ion-app>
</template>
