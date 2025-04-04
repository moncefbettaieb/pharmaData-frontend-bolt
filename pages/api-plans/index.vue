<template>
  <div class="bg-white">
    <div class="mx-auto max-w-7xl px-6 lg:px-8">
      <div class="mx-auto max-w-4xl text-center">
        <h1 class="text-base font-semibold leading-7 text-indigo-600">Tarification</h1>
        <p class="mt-2 text-4xl font-bold tracking-tight text-gray-900 sm:text-5xl">
          Plans d'accès à l'API
        </p>
        <p class="mt-6 text-lg leading-8 text-gray-600">
          Choisissez le plan qui correspond à vos besoins. Tous les prix sont en euros et par mois.
        </p>
      </div>

      <div class="isolate mx-auto mt-16 grid max-w-md grid-cols-1 gap-y-8 sm:mt-20 lg:mx-0 lg:max-w-none lg:grid-cols-3">
        <div
          v-for="plan in plansStore.plans"
          :key="plan.id"
          class="flex flex-col justify-between rounded-3xl bg-white p-8 ring-1 ring-gray-200 xl:p-10"
          :class="{
            'lg:z-10 lg:rounded-b-none': plan.id === 'pro',
            'lg:mt-8': plan.id !== 'pro'
          }"
        >
          <div>
            <div class="flex items-center justify-between gap-x-4">
              <h2 class="text-lg font-semibold leading-8 text-gray-900">
                {{ plan.name }}
              </h2>
            </div>
            <p class="mt-4 text-sm leading-6 text-gray-600">
              {{ plan.requestsPerMonth.toLocaleString() }} requêtes par mois
            </p>
            <p class="mt-6 flex items-baseline gap-x-1">
              <span class="text-4xl font-bold tracking-tight text-gray-900">{{ plan.price }}€</span>
              <span class="text-sm font-semibold leading-6 text-gray-600">/mois</span>
            </p>
            <ul role="list" class="mt-8 space-y-3 text-sm leading-6 text-gray-600">
              <li v-for="feature in plan.features" :key="feature" class="flex gap-x-3">
                <svg class="h-6 w-5 flex-none text-indigo-600" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                  <path fill-rule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z" clip-rule="evenodd" />
                </svg>
                {{ feature }}
              </li>
            </ul>
          </div>
          <button
            @click="selectPlan(plan)"
            :disabled="!user || loading"
            class="mt-8 block rounded-md bg-indigo-600 px-3 py-2 text-center text-sm font-semibold leading-6 text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <span v-if="loading">Chargement...</span>
            <span v-else>{{ user ? 'Sélectionner ce plan' : 'Connectez-vous pour souscrire' }}</span>
          </button>
        </div>
      </div>

      <div v-if="!user" class="mt-8 text-center">
        <p class="text-gray-600">
          Vous devez être connecté pour souscrire à un plan.
          <NuxtLink to="/login" class="text-indigo-600 hover:text-indigo-500">
            Se connecter
          </NuxtLink>
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { useAuthStore } from '~/stores/auth'
import { usePlansStore } from '~/stores/plans'
import { useToast } from 'vue-toastification'
import { loadStripe } from '@stripe/stripe-js'
import { httpsCallable } from 'firebase/functions'
import { storeToRefs } from 'pinia'

const authStore = useAuthStore()
const plansStore = usePlansStore()
const { user } = storeToRefs(authStore)
const toast = useToast()
const config = useRuntimeConfig()
const loading = ref(false)
const { $functions } = useNuxtApp()

const selectPlan = async (plan) => {
  if (!user.value) {
    toast.error('Veuillez vous connecter pour souscrire à un plan')
    return
  }

  loading.value = true
  try {
    if (!$functions) {
      throw new Error('Firebase Functions non initialisé')
    }

    // Vérifier que la clé Stripe est disponible
    console.log('Clé publique Stripe:', config.public.stripePublicKey)
    console.log('Clé secrete Stripe:', config.public.stripeSecretKey)
    if (!config.public.stripePublicKey) {
      console.error('Clé publique Stripe manquante dans la configuration')
      throw new Error('Configuration Stripe manquante')
    }
    
    console.log('Clé publique Stripe:', config.public.stripePublicKey)

    // Initialiser Stripe
    const stripe = await loadStripe(config.public.stripePublicKey)
    if (!stripe) {
      throw new Error('Erreur lors du chargement de Stripe')
    }

    // Appeler la Cloud Function
    const createSubscriptionCall = httpsCallable($functions, 'createSubscription')
    
    console.log('Appel de la fonction avec les paramètres:', {
      priceId: plan.id,
      successUrl: `${window.location.origin}/payment/success`,
      cancelUrl: `${window.location.origin}/payment/cancel`
    })
    
    const result = await createSubscriptionCall({
      priceId: plan.id,
      successUrl: `${window.location.origin}/payment/success`,
      cancelUrl: `${window.location.origin}/payment/cancel`
    })

    console.log('Résultat de la fonction:', result)
    
    const { sessionId } = result.data
    if (!sessionId) {
      console.error('Réponse de la fonction:', result.data)
      throw new Error('Session ID manquant dans la réponse')
    }

    // Rediriger vers Stripe Checkout
    const { error } = await stripe.redirectToCheckout({ sessionId })
    if (error) {
      console.error('Erreur Stripe:', error)
      throw error
    }
  } catch (error) {
    console.error('Erreur complète:', error)
    toast.error("Une erreur s'est produite lors de la redirection vers le paiement")
  } finally {
    loading.value = false
  }
}
</script>