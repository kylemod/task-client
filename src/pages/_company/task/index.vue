<template>
  <div>
    <div class="flex justify-between">
    <p class="text-2xl font-bold text-gray-600">Task</p>
    </div>
    
    <div class="flex flex-wrap mt-4 sm:gap-x-2 gap-y-2 ">
      
      <div v-if="fetchingAllTask">
        <p class="py-4 text-base text-center text-gray-500 font-medium"><ArrowPathIcon class="h-6 w-6 inline mr-2 animate-spin"></ArrowPathIcon> Fetching Task...</p>
      </div>
      <router-link v-else :to="`/${route.params.company}/task/${task._id}`" v-for="task in tasks" :key="task.id" class="block bg-white shadow px-4 py-4 w-full sm:w-52 rounded-md">
        <div class="flex justify-between">
          <p class="text-gray-600 font-medium">{{ task.title }}</p>
        </div>
        <p class="text-sm text-gray-400 mt-0.5 mb-4">{{ task.workspace.title }}</p>
        <span v-if="!task.todos.every(d => d.done === true) && !task.status">
          <p class="inline-block py-1 px-2 bg-red-100 text-red-400 w-50 text-xs rounded-md font-medium mb-1.5">{{ task.date }}</p>
        </span>
        <span v-else>
        <p v-if="(task.todos.filter(tk => tk.done)).length == task.todos.length" class="inline-block py-1 px-2 bg-emerald-100 text-emerald-400 w-50 text-xs rounded-md font-medium mb-4">{{ task.date }}</p>
        <p v-else-if="(task.todos.filter(tk => tk.done)).length != 0" class="inline-block py-1 px-2 bg-orange-100 text-orange-400 w-50 text-xs rounded-md font-medium mb-4">{{ task.date }}</p>
        <p v-else class="inline-block py-1 px-2 bg-gray-100 text-gray-400 w-50 text-xs rounded-md font-medium mb-4">{{ task.date }}</p>
        </span>
        <p class="inline-block py-1 px-2 bg-cyan-100 text-cyan-600 w-50 text-xs rounded-md font-medium mb-4 ml-2">{{ task.dueDate }}</p>
        <div class="flex justify-between mb-2">
          <p class="text-gray-600 text-sm">
            <span v-if="!task.todos.every(d => d.done === true) && !task.status">
              <span>Failed</span>
            </span>
            <span v-else>
              <span v-if="(task.todos.filter(tk => tk.done)).length == task.todos.length">
                Completed
              </span>
              <span v-else-if="(task.todos.filter(tk => tk.done)).length != 0">
                In Progress
              </span>
              <span v-else>New</span>
            </span>
          </p>
          <p class="text-gray-400 font-medium text-sm"> {{ (task.todos.filter(tk => tk.done == true)).length }} / {{ task.todos.length }}</p>
        </div>
        <div class="w-full bg-gray-200 rounded-full h-1">
          <div v-if="!task.todos.every(d => d.done === true) && !task.status" class="h-1 rounded-full bg-red-500" :style="{ 'width': (task.todos.filter(tk => tk.done == true)).length / task.todos.length * 100 + '%' }"></div>
          <div v-else class="h-1 rounded-full " :class="[(task.todos.filter(tk => tk.done == true)).length != task.todos.length ? 'bg-orange-500': '', (task.todos.filter(tk => tk.done == true)).length == task.todos.length ? 'bg-emerald-500': '']" :style="{ 'width': (task.todos.filter(tk => tk.done == true)).length / task.todos.length * 100 + '%' }"></div>
        </div>
        </router-link>
      
    </div>
  </div>
</template>

<script setup>
import { EllipsisVerticalIcon, ArrowPathIcon } from '@heroicons/vue/24/outline'
import { Dialog, DialogPanel, DialogTitle, TransitionChild, TransitionRoot } from '@headlessui/vue'
import axios from 'axios'
import { useUserStore } from '@/stores/user'
import { useToast } from 'vue-toast-notification'

const $toast = useToast()
const router = useRouter()
const route = useRoute()
const user = useUserStore()

onBeforeMount(async() => {
  await axios.post('api/user/get-superadmins', {
    company: route.params.company
  })
  .then(res => {
    if(res.data.length == 0) {
      $toast.error(`No company named ${route.params.company} exist!`, {position: 'top-right'})
      router.back()
    }
  })
  .catch(err => {
    console.log(err)
  })
  if(user.role == 'admin' || user.role == 'superadmin') {
    $toast.info('User redirected as admin!', {position: 'top-right'})
    router.push(`/${route.params.company}/admin/task`)
  } else {
    await axios.post('api/auth/get-session', {
      userId: user.token
    })
      .then(res => {
        let auth = res.data.authenticated
        let userData = res.data.user
        let token = res.data.token
        if(auth) {
          user.setUser({ userData, auth, token })
          console.log('auth')
        }
      })
      .catch(err => {
        user.$reset()
        router.push(`/login`)
        $toast.error('User not authenticated!', {position: 'top-right'})
        console.log('not auth')
      })
  }
})

const fetchingAllTask = ref(true)

const tasks = ref()

let validateDate = (dueDate, taskTitle) => {
  let due = new Date(`${dueDate}`)
  let today = new Date()
  let timeDiff = due.getTime() - today.getTime()
  
  let remaining = Math.ceil(timeDiff / (1000 * 3600 * 24))
  let notifThreshold = 1
  //console.log(remaining)
  
  if(remaining <= 1 && remaining > 0) {
    console.log('due date is approaching')
    $toast.info(`${taskTitle} due date is approaching!`, {position: 'top-right'})
  } else if(remaining <= 0) {
    //$toast.warning(`${taskTitle} due date has passed!`, {position: 'top-right'})
    console.log('duedate has passed')
  } else {
    console.log('check your tasks!')
  }
  
}

onMounted(async () => {
  await axios.post('api/task/user/get-task', {
    userId: user.id
  })
    .then(res => {
      tasks.value = res.data
      tasks.value.reverse()
      
      tasks.value.forEach(t => {
        validateDate(t.dueDate, t.title)
      })
      
      console.log(tasks.value)
    })
    .catch(err => {
      $toast.error('An error occured', {position: 'top-right'})
      console.log(err)
    })
  fetchingAllTask.value = false
  console.log(user)
})

</script>

<route lang="yaml">
meta:
  layout: users
</route>