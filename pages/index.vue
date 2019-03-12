<template lang="pug">
div
  section.section(v-if="drained")
    .container
      .notification.is-warning This faucet has been drained. Sorry for inconvenience.

  main.section
    form(@submit.prevent="claim")
      .container
        .columns
          .column.is-8
            b-field(label="Recipient")
              b-input(v-model="form.recipient"
                required
                :pattern="recipientPattern"
                :title="recipientPlaceholder"
                :placeholder="recipientPlaceholder"
                maxlength="46"
                :disabled="drained"
              )
          .column.is-4
            b-field(label="Amount")
              b-input(v-model.number="form.amount" type="number"
                :placeholder="amountPlaceholder"
                min="0"
                :max="outOpt"
                :step="step"
                :disabled="drained"
              )

        .columns
          .column.is-8
            b-field(label="Message")
              b-input(v-model="form.message"
                maxlength="1023"
                placeholder="(Optional)"
                :disabled="drained"
              )
          .column.is-4
            b-field(label="Submit")
              button(type="submit" class="button is-primary is-fullwidth" :disabled="drained || waiting")
                span CLAIM!

        .columns
          .column.is-8
            b-field(label="Faucet Address")
              b-input(:value="address" readonly :disabled="drained")
          .column.is-4
            b-field(label="Faucet Balance")
              b-input(:value="balance" type="number" readonly :disabled="drained")

  Readme(
    :publicUrl="publicUrl"
    :network="network"
    :mosaicFqn="mosaicFqn"
    :outMin="outMin"
    :outMax="outMax"
  )
</template>

<script>
import qs from 'querystring'
import Readme from '~/components/Readme'
import axios from 'axios'

export default {
  name: 'Home',
  components: {
    Readme
  },
  data() {
    return {
      form: {
        recipient: null,
        message: null,
        amount: null
        // encrypted: false,
        // asMosaic: false
      },
      drained: true,
      waiting: false,
      network: null,
      apiUrl: null,
      publicUrl: null,
      mosaicFqn: null,
      outMin: null,
      outMax: null,
      outOpt: null,
      step: null,
      address: null,
      balance: null,
      recipientPattern: null,
      recipientPlaceholder: null,
      amountPlaceholder: null
    }
  },
  asyncData({ res, error }) {
    if (res.data.error) {
      return error(res.data.error)
    }
    const attrs = res.data.data.attributes
    const firstChar = attrs.address[0]
    const recipientPattern = `^${firstChar}[ABCD].+`
    const recipientPlaceholder = `${
      attrs.network
    } address start with a capital ${firstChar}`
    const amountPlaceholder = `(Up to ${
      attrs.outOpt
    }. Optional, if you want fixed amount)`
    const data = {
      ...attrs,
      recipientPattern,
      recipientPlaceholder,
      amountPlaceholder
    }
    console.debug('asyncData: %o', data)
    return data
  },
  async mounted() {
    await this.$recaptcha.init()
    if (process.browser) {
      /* eslint nuxt/no-globals-in-created: 0 */
      const params = qs.parse(window.location.search.substring(1))
      this.form.recipient = params.recipient
      this.form.amount = params.amount
      this.form.message = params.message
    }
  },
  methods: {
    async claim() {
      this.waiting = true
      this.$router.push({ path: this.$route.path, query: this.form })
      const reCaptcha = await this.$recaptcha.execute('login')
      axios
        .post('/claims', { reCaptcha, ...this.form })
        .then(response => {
          this.info(`Send your declaration.`)
          this.success(`Transaction Hash: ${response.data.txHash}`)
        })
        .catch(err => {
          console.debug(err.message)
          const msg =
            (err.response.data && err.response.data.error) ||
            err.response.statusTest
          this.failed(`Message from server: ${msg}`)
        })
        .finally(() => {
          this.waiting = false
        })
    },
    info(message) {
      this.$snackbar.open({
        type: 'is-info',
        message,
        queue: false
      })
    },
    success(message) {
      this.$snackbar.open({
        type: 'is-success',
        message,
        queue: false,
        duration: 8000
      })
    },
    warning(message) {
      this.$snackbar.open({
        type: 'is-warning',
        message,
        queue: false,
        duration: 8000
      })
    },
    failed(message) {
      this.$snackbar.open({
        type: 'is-danger',
        message,
        queue: false,
        duration: 8000
      })
    }
  }
}
</script>