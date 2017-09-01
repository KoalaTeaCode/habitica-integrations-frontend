<template lang="pug">
  div
    h1 Welcome to Habitica Integrations
    transition(name="fade")
      div(v-if='page === "login"')
        h2 Login to Habitica
        form.col-6.offset-3
          .form-group
            label Habitica ID
            input.form-control(type='text', aria-describedby='emailHelp', placeholder='Enter Habitica ID', v-model='habiticaInfo.userId')
          .form-group
            label API Key
            input.form-control(type='text', aria-describedby='emailHelp', placeholder='Enter Habitica API Key', v-model='habiticaInfo.apikey')
          button.btn.btn-primary(@click.prevent='login()') Add Habitica Account

    transition(name="fade")
      div(v-if='page === "action"')
        .col-6.offset-3
          div
            h2 Select an Action
            small Select a Service coming soon...

          div
            select.form-control()
              option Get points when I archive a Card
              //- option Sync a Todo With a List
            button.btn.btn-primary(@click.prevent='changePage("trello")') Login

          div(v-if='actions.length > 0', style='margin-top: 1em;')
            h3 Current Actions
            ul
              li(v-for='action in actions') {{action.type}} {{action.idModel}}

    transition(name="fade")
      div(v-if='page === "trello"')
        .col-6.offset-3
          button.btn.btn-primary(@click='loginToTrello()') Login With Trello
          .form-group(v-if='loggedInWithTrello')
            label Select a Board
            select.form-control(v-model='selectedBoard', @change='getLists()')
              option(v-for='board in boards', :value='board.id') {{board.name}}
          .form-group(v-if='selectedBoard')
            label Select a List
            select.form-control(v-model='selectedList')
              option(v-for='list in lists', :value='list.id') {{list.name}}
          .form-group(v-if='selectedList')
            button.btn.btn-primary(@click.prevent='addAction()', v-if='!loading') Add Action
</template>

<script>
import axios from 'axios'
// const API_URL = 'http://localhost:3000/api/v1/'
const API_URL = 'https://habitica-integration-api.herokuapp.com/api/v1/'

export default {
  name: 'hello',
  data () {
    return {
      loading: false,
      habiticaInfo: {
        apikey: '',
        userId: ''
      },
      page: 'login',
      boards: [],
      selectedBoard: '',
      lists: [],
      selectedList: '',
      loggedInWithTrello: false,
      actions: []
    }
  },
  mounted () {
    axios.defaults.baseURL = API_URL

    let token = localStorage.getItem('user-token')
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`

      axios.get('actions')
      .then(response => {
        this.actions = response.data.actions
        this.changePage('action')
      })
      .catch(error => {
        // Assume token expired
        localStorage.setItem('user-token', '')
        axios.defaults.headers.common['Authorization'] = ''
        console.log(error.message)
      })
    };
    // @TODO: get actions
  },
  methods: {
    addAction () {
      let idModel = this.selectedList
      let callbackURL = API_URL + 'trello'
      this.loading = true;

      window.Trello.post(`/webhooks/?idModel=${idModel}&callbackURL=${callbackURL}`,
        (response) => {
          axios.post('actions', {
            type: 'trello',
            idModel: this.selectedList
          })
          .then(response => {
            alert(response.data.message)
            this.loading = false;
            this.changePage('action')
          })
          .catch(error => {
            this.loading = false;
            alert(error.message)
          })
        },
        (response) => {
          this.loading = false;
          alert('Error')
        })
    },
    login () {
      if (!this.habiticaInfo.apikey || !this.habiticaInfo.userId) {
        alert('You must enter your API Key and User ID')
        return
      }

      axios.post('authenticate', {
        apiKey: this.habiticaInfo.apikey,
        userId: this.habiticaInfo.userId
      })
      .then(result => {
        localStorage.setItem('user-token', result.data.token)
        this.changePage('action')
      })
      .catch(err => {
        alert(err.message)
      })
    },
    changePage (newPage) {
      this.page = newPage
    },
    getLists () {
      if (!this.selectedBoard) return
      window.Trello.boards.get(`${this.selectedBoard}/lists`)
        .then((result) => {
          this.lists = result
        })
    },
    authenticationSuccess (one, two) {
      this.loggedInWithTrello = true

      window.Trello.members.get('/me')
        .then((result) => {
          console.log(result)
        })

      window.Trello.members.get('/me/boards')
        .then((result) => {
          this.boards = result
        })
    },
    authenticationFailure () {
      alert('Failed authentication')
    },
    loginToTrello () {
      window.Trello.authorize({
        type: 'popup',
        name: 'Getting Started Application',
        scope: {
          read: 'true',
          write: 'true' },
        expiration: 'never',
        success: this.authenticationSuccess,
        error: this.authenticationFailure
      })
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  h1, h2 {
    font-weight: normal;
  }

  h1 {
    margin-bottom: 1em;
  }

  ul {
    list-style-type: none;
    padding: 0;
  }

  li {
    display: inline-block;
    margin: 0 10px;
  }

  a {
    color: #42b983;
  }

  .fade-enter-active, .fade-leave-active {
    transition: opacity .5s
  }
  .fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
    opacity: 0
  }
</style>
