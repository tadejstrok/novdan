<template>
  <div class="payment">
    <h3>PLAČILO</h3>
    <p>
      Vpiši podatke za plačilo. Znesek bo obračunan danes in nato vsakega 1. v
      mesecu. Naročnino lahko kadarkoli prekineš.
    </p>
    <div v-if="error" class="alert alert-danger">
      <h4>Napaka št. {{ error.status }}</h4>
      <p>
        Naš strežnik je ni mogel
        rešiti, prejel je naslednje sporočilo:
        <strong>{{
          error.data && error.data.msg ? error.data.msg : error.message
        }}</strong>
      </p>
      <p>
        Zaračunali ti nismo ničesar, ves denar je še vedno na tvoji kartici.
        Predlagamo, da osvežiš stran in poskusiš ponovno. Če ne bo šlo, nam piši
        na
        <a href="mailto:vsi@danesjenovdan.si">vsi@danesjenovdan.si</a> in ti
        bomo poskusili pomagati.
      </p>
    </div>
    <loading v-if="paymentInProgress || loading" />

    <div class="buttons-wrapper links">
      <button
        :class="{ active: paymentType == 'card' }"
        @click="paymentType = 'card'; error = null"
      >
        Kreditna kartica
      </button>
      <button
        :class="{ active: paymentType == 'paypal' }"
        @click="paymentType = 'paypal'; error = null"
      >
        Paypal
      </button>
    </div>

    <Card
      v-if="token && paymentType == 'card'"
      :token="token"
      @ready="onPaymentReady"
      @validity-change="paymentInfoValid = $event"
      @payment-start="paymentInProgress = true"
      @success="paymentSuccess"
      @error="paymentError"
    />

    <Paypal
      v-if="token && paymentType == 'paypal'"
      :token="token"
      :amount="5"
      :recurring="true"
      @ready="onPaymentReady"
      @payment-start="paymentInProgress = true"
      @success="paymentSuccess"
    />

    <div class="buttons-wrapper">
      <button :disabled="!canContinueToNextStage" @click="finish">
        Plačaj
      </button>
    </div>
    <div class="buttons-wrapper">
      <nuxt-link to="/dash">
        Nazaj
      </nuxt-link>
    </div>
  </div>
</template>

<script>
import api from '~/mixins/api.js'

export default {
  components: {
    // PaymentError
  },
  mixins: [api],
  layout: 'login',
  data() {
    return {
      token: null,
      paymentInfoValid: false,
      paymentInProgress: false,
      paymentType: 'card',
      payFunction: undefined,
      nonce: undefined,
      error: null,
      loading: true
    }
  },
  computed: {
    canContinueToNextStage() {
      return this.payFunction && this.paymentInfoValid
    }
  },
  async mounted() {
    if (!this.$api.hasToken()) {
      this.$router.replace('/dash/login')
    }
    try {
      this.status = await this.$api.getStatus()
      console.log(this.status)
      const response = await this.$api.activateSubscription()
      this.token = response.token
      this.loading = true
    } catch (e) {
      this.$router.replace('/dash/login')
    }
  },
  methods: {
    onPaymentReady({ pay } = {}) {
      this.checkoutLoading = false
      this.paymentInfoValid = false
      this.payFunction = pay
      this.loading = false
    },
    finish() {
      this.error = null
      console.log('pritisnemo gumb')
      if (this.payFunction) {
        console.log('pay funcrion')
        this.payFunction()
      }
    },
    async paymentSuccess({ nonce } = {}) {
      this.paymentInProgress = true
      this.nonce = nonce
      try {
        const response = await this.$api.activateSubscription2(this.nonce)
        console.log(response)
        this.$router.push('/dash')
        // this.$router.push(
        //   response.upload_token
        //     ? `/doniraj/hvala?token=${response.upload_token}`
        //     : '/doniraj/hvala'
        // )
        // this.paymentInProgress = true
      } catch (error) {
        this.paymentInProgress = false
        // eslint-disable-next-line no-console
        console.error(error.response)
        this.error = error
      }
    },
    paymentError(argument) {
      this.paymentInProgress = false
      // eslint-disable-next-line
      console.log('ERROR VERY ERROR')
      // eslint-disable-next-line
      console.log(argument)
      this.error = argument.error
    }
  }
}
</script>

<style lang="scss" scoped>
h3 {
  text-align: center;
}
.alert-danger {
  border: 2px solid red;
  padding: 0 16px;
  h4, strong {
    color: red;
  }
}
.buttons-wrapper {
  text-align: center;
  margin: 20px 0;
  button {
    border: 2px solid #000000;
    border-radius: 10px;
    background-color: #ffffff;
    font-family: 'Syne', sans-serif;
    font-size: 25px;
    font-weight: 700;
    padding: 8px 24px;
    margin: 10px 0;
    // &.active {
    //   background-color: #ffd700;
    // }
    &:not([disabled]):hover {
      background-color: #1103b1;
      border-color: #1103b1;
      color: white;
      cursor: pointer;
    }
  }
}
.buttons-wrapper.links {
  button {
    border: none;
    border-radius: 0;
    background-color: transparent;
    font-size: 18px;
    font-weight: 400;
    padding: 8px;
    &.active {
      background-color: transparent;
      text-decoration: underline;
    }
    &:hover {
      background-color: #1103b1;
      border-color: #1103b1;
      color: white;
    }
  }
}
</style>
