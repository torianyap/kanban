<template>
  <div>
    <NavBar
      :loggedIn='loggedIn'
      @changePage='changePage'
      @logout='logout'>
    </NavBar>
    <RegisterPage 
      v-if="pageName === 'register-page'"
      @googleSignIn='googleSignIn'
      @register="register">
    </RegisterPage>
    <LoginPage 
      v-else-if="pageName === 'login-page'"
      @login="login"
      @googleSignIn='googleSignIn'>
    </LoginPage>
    <HomePage 
      v-else-if="pageName === 'home-page'"
      :kanban="kanban"
      :loggedIn='loggedIn'
      @fetchKanban="fetchKanban"
      @changeCategory="changeCategory"
      @changePage='changePage'
      @deleteTask='deleteTask'
      @updateTask='updateTask'>
    </HomePage>
    <AddPage 
      v-else-if="pageName === 'add-page'"
      @changePage='changePage'
      @addTask='addTask'>
    </AddPage>
  </div>
</template>

<script>
const Toast = Swal.mixin({
  toast: true,
  position: 'top-end',
  showConfirmButton: false,
  timer: 3000,
  timerProgressBar: true,
  didOpen: (toast) => {
    toast.addEventListener('mouseenter', Swal.stopTimer)
    toast.addEventListener('mouseleave', Swal.resumeTimer)
  }
})
import NavBar from './components/NavBar'
import RegisterPage from './components/RegisterPage'
import LoginPage from './components/LoginPage'
import HomePage from './components/HomePage'
import AddPage from './components/AddPage'
import axios from './config/axios'

export default {
  name: 'App',
  data() {
    return {
        pageName: 'register-page',
        loggedIn: {
            status: false,
            user: null,
            username: null
        },
        kanban: []
    }
  },
  components: {
    RegisterPage, LoginPage, HomePage, AddPage, NavBar
  },
  methods: {
    changePage(name) {
        this.pageName = name
    },
    logout() {
        Swal.fire({
            title: 'Logging out',
            text: "You will be logged out",
            icon: 'warning',
            showCancelButton: true,
            confirmButtonText: 'Logout'
        }).then((result) => {
            if (result.isConfirmed) {
                localStorage.clear()
                this.pageName = 'login-page'
                this.loggedIn.status = false
                this.loggedIn.user = ''
                Swal.fire({
                    icon: 'success',
                    title: 'logged out'
                })
            }
        })
    },
    login(payload) {
        axios({
            method: 'POST',
            url: `/login`,
            data: {
                email: payload.email,
                password: payload.password
            }
        })
        .then(response => {
            const access_token = response.data.access_token
            const user = response.data.user
            this.pageName = 'home-page'

            localStorage.setItem('access_token', access_token)
            localStorage.setItem('user', JSON.stringify(user, null, 2))

            this.loggedIn.status = true
            this.loggedIn.user = JSON.parse(localStorage.user)
            this.loggedIn.username = JSON.parse(localStorage.user).username

            Toast.fire({
                icon: 'success',
                title: `Welcome ${user.username}`
            })
            this.fetchKanban()
        })
        .catch(err => {
            Swal.fire({
                icon: 'error',
                title: 'Something went wrong',
                text: err.response.data.msg
            })
        })
    },
    register(payload) {
        axios({
            method: 'POST',
            url: `/register`,
            data: payload
        })
        .then(response => {
            const access_token = response.data.access_token
            const user = response.data.user

            localStorage.setItem('access_token', access_token)
            localStorage.setItem('user', JSON.stringify(user, null, 2))

            this.pageName = 'home-page'

            this.loggedIn.status = true
            this.loggedIn.user = JSON.parse(localStorage.user)
            this.loggedIn.username = JSON.parse(localStorage.user).username

            Toast.fire({
                icon: 'success',
                title: `Welcome ${user.username}`
            })
        })
        .catch(err => {
            Swal.fire({
                icon: 'error',
                title: 'Something went wrong',
                text: err.response.data.msg
            })
        })  
    },
    googleSignIn(token){
        axios({
            method: 'POST',
            url: '/googleLogin',
            data: {
                google_access_token: token
            }
        })
        .then(({data}) => {
            const access_token = data.access_token
            const user = data.user
            
            localStorage.setItem('access_token', access_token)
            localStorage.setItem('user', JSON.stringify(user, null, 2))
            
            this.pageName = 'home-page'

            this.loggedIn.status = true
            this.loggedIn.user = JSON.parse(localStorage.user)
            this.loggedIn.username = JSON.parse(localStorage.user).username

            Toast.fire({
                icon: 'success',
                title: `Welcome ${user.username}`
            })
            this.fetchKanban()
        })
        .catch(err => {
            console.log(err)
        })
    },
    fetchKanban() {
        const access_token = localStorage.getItem('access_token')
        axios({
            method: 'GET',
            url: '/tasks',
            headers: {
                access_token
            }
        })
        .then(response => {
            this.kanban = response.data
        })
        .catch(err => {
            console.log(err, '<<< ini error fetch')
        })
    },
    changeCategory(payload) {
        const access_token = localStorage.getItem('access_token')
        axios({
            method: 'PATCH',
            url: `/tasks/${payload.id}`,
            headers: {
                access_token
            },
            data: {
                category: payload.category
            }
        })
        .then(({data}) => {
            Toast.fire({
                icon: 'success',
                title: `${data.title} moved to ${payload.category}`
            })
            this.$socket.emit('changeData')
            this.fetchKanban()
        })
        .catch(err => {
            Swal.fire({
                icon: 'error',
                title: err.response.data.msg
            })
        })
    },
    addTask(payload) {
        const access_token = localStorage.getItem('access_token')
        axios({
            method: 'POST',
            url: '/tasks',
            headers: {
                access_token
            },
            data: payload
        })
        .then(response => {
            this.pageName = 'home-page'
            this.fetchKanban()
            this.$socket.emit('changeData')
        })
        .catch(err => {
            Swal.fire({
                icon: 'error',
                title: 'Something went wrong',
                text: err.response.data.msg
            })
        })
    },
    deleteTask(id){
        Swal.fire({
            title: 'Are You Sure ?',
            text: 'This task will be deleted',
            icon: 'warning',
            showCancelButton: true,
            confirmButtonText: 'Delete'
        })
        .then(result => {
            if (result.isConfirmed) {
                const access_token = localStorage.getItem('access_token')
                axios({
                    method: 'DELETE',
                    url: `tasks/${id}`,
                    headers: {
                        access_token
                    }
                })
                .then(response => {
                    this.fetchKanban()
                    Toast.fire({
                        icon: 'success',
                        title: response.data.msg
                    })
                    this.$socket.emit('changeData')
                })
                .catch(err => {
                    Swal.fire({
                        icon: 'error',
                        title: err.response.data.msg
                    })
                })
            }
        })
    },
    updateTask(payload){
        Swal.fire({
            title: 'Edit Task',
            html: `<input type="text" id="title" class="swal2-input" value="${payload.task.title}" required>
            <input type="text" id="description" class="swal2-input" value="${payload.task.description}" required>`,
            confirmButtonText: 'Edit Task',
            showCancelButton: true,
            reverseButtons: true,
            preConfirm: () => {
                const title = Swal.getPopup().querySelector('#title').value
                const description = Swal.getPopup().querySelector('#description').value
                if (!title || !description) {
                    Swal.showValidationMessage('Please complete the form')
                }
                return {title, description}
            }
        })
        .then(result => {
            if (result.isConfirmed) {
                const access_token = localStorage.getItem('access_token')
                axios({
                    method: 'PUT',
                    url: `/tasks/${payload.id}`,
                    headers: {
                        access_token
                    },
                    data: {
                        title: result.value.title,
                        description: result.value.description
                    }
                })
                .then(response => {
                    this.$socket.emit('changeData')
                    this.fetchKanban()
                    Toast.fire({
                        icon: 'success',
                        title: `${payload.task.title} has been updated`
                    })
                })
                .catch(err => {
                    Swal.fire({
                        icon: 'error',
                        title: 'Something went wrong',
                        text: err.response.data.msg
                    })
                })
            }
        })
    },
  },
  created() {
      if (localStorage.access_token) {
          this.pageName = 'home-page'
          this.loggedIn.status = true
          this.loggedIn.user = JSON.parse(localStorage.user)
          this.loggedIn.username = JSON.parse(localStorage.user).username

          this.fetchKanban()
      } else {
          this.pageName = 'register-page'
      }
  },
  sockets: {
    changeData (tasks) {
      this.kanban = tasks
    }
  }
}
</script>

<style>

</style>