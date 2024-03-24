<template>
  <q-page class="q-pa-md">
    <q-list bordered class="rounded-borders q-mb-md">
      <q-item 
        v-for="task in tasks" 
        :key="task.id" 
        clickable 
        v-ripple
        @click.stop="openTaskDialog(task)">
        <q-item-section>
          <q-item-label>{{ task.name }}</q-item-label>
          <q-item-label caption>{{ task.description }}</q-item-label>
        </q-item-section>
        <q-item-section side top>
          <q-item-label caption>{{ task.dueTime }}</q-item-label>
        </q-item-section>
        <q-item-section side top>
          <q-btn 
            flat 
            round 
            dense 
            icon="delete" 
            @click="removeTask(task.id)" />
        </q-item-section>
      </q-item>
    </q-list>

    <!-- Details Dialog-->
    <q-dialog v-model="taskDialog" full-width full-height>
      <q-card class="full-height">
        <q-bar>
          <q-btn dense flat icon="close" @click="closeTaskDialog();"/>
          <q-space/>
        </q-bar>
        <q-card-section class="q-pt-none">
          <div class="text-h6" style="margin-bottom: 10px">{{ selectedTask?.name }}</div>
          <p v-html="formatDescription(selectedTask?.description)"></p>
          <div v-if="selectedTask?.dueTime" style="margin-bottom: 20px;">Due Time: {{ selectedTask?.dueTime }}</div>
        </q-card-section>

        <q-card-actions align="around">
          <q-btn flat label="Completed" color="primary" @click="completeTask(selectedTask.id);"/>
          <q-btn flat label="Close" color="primary" @click="closeTaskDialog();"/>
        </q-card-actions>
      </q-card>
    </q-dialog>
    
    <!-- Task Dialog-->
    <q-dialog v-model="dialog" full-width full-height>
      <q-card>
        <q-card-section class="row items-center">
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>
        <q-card-section>
          <div class="text-h6" style="margin-bottom: 20px">Add New Task</div>
          <q-input 
            v-model="newTask.name" 
            :error="nameError" 
            :error-message="nameErrorMessage" 
            label="Task Name" 
            filled 
            @input="validateName"/>
          <q-input 
            v-model="newTask.description" 
            label="Description" 
            type="textarea" filled />
          <q-input 
            filled 
            readonly 
            v-model="newTask.dueTime" 
            label="Due Time" 
            hint="Tap to set time" 
            @click="timeDialog = true" />
        </q-card-section>
        <q-card-actions align="right">
          <q-btn label="Add" color="primary" @click="addTask();" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Time Picker-->
    <q-dialog v-model="timeDialog">
      <q-card>
        <q-card-section class="row items-center">
          <q-time v-model="time" format24h />
          <q-btn label="Set Time" color="primary" @click="setTime();" />
        </q-card-section>
      </q-card>
    </q-dialog>

    <div class="absolute-bottom-right">
      <q-btn fixed fab icon="add" color="green" bottom right @click="openDialog();"/>
    </div>
  </q-page>
</template>

<script>
import {LocalNotifications} from '@capacitor/local-notifications';

export default {
  data() {
    return {
      tasks: [],
      dialog: false,
      timeDialog: false,
      time: '12:00',
      newTask: {
        name: '',
        description: '',
        dueTime: '',
      },
      nameError: false,
      nameErrorMessage: 'Task name is required',
      taskDialog: false,
      selectedTask: null,
    };
  },
  methods: {
    openDialog() {
      this.dialog = true;
    },
    addTask() {
      if(this.validateName()){
        const newTaskID = Math.floor(Date.now() / 1000);

        const newTask = {
          ...this.newTask,
          id: newTaskID
        };
        
        console.log('New Task: ', newTask);
        this.tasks.push(newTask);
        
        if(newTask.dueTime){
          this.scheduleNotification(newTask);
        }

        this.resetNewTask();
        this.dialog = false;

        navigator.notification.alert(
          'Task has been added',
          () => {},
          `${newTask.name} is added`,
          'OK'
          )
          
      }
    },
    removeTask(id) {
      this.tasks = this.tasks.filter(task => task.id !== id);
    },
    setTime() {
      this.newTask.dueTime = this.time;
      this.timeDialog = false;
      console.log('Due Time: ',this.newTask.dueTime);
    },
    resetNewTask() {
      this.newTask = { name: '', description: '', dueTime: '' };
      this.time = '12:00'; // Reset time for the next use
    },
    validateName(){
      this.nameError = !this.newTask.name.trim();
      return !this.nameError;
    },
    openTaskDialog(task){
      this.selectedTask = task;
      this.taskDialog = true;
    },
    closeTaskDialog(){
      this.taskDialog = false;
      this.selectedTask = null;
    },
    formatDescription(description){
      return description.replace(/\n/g, '<br>');
    },
    completeTask(id){
      this.removeTask(id);
      this.closeTaskDialog();
    },
    async scheduleNotification(task){
      const [hours, minutes] = task.dueTime.split(':').map(Number);
      const now = new Date();
      const dueTimeDate = new Date(now.getFullYear(), now.getMonth(), now.getDate(), hours, minutes);
      const notificationTime = new Date(dueTimeDate.getTime() - 1 * 60000);

      //const name = task.name;

      // console.log(`Schedule task.dueTime: `, task.dueTime);
      // console.log(`Schedule dueTimeDate: `, dueTimeDate);
      // console.log('Task ID: ', task.id);
      
      await LocalNotifications.schedule({
        notifications: [
          {
            title: `Upcoming Task: ${task.name}`,
            body: 'This task is due in 1 minute.',
            id: task.id,
            schedule: { at: notificationTime },
          },
        ],
      });
      // navigator.notification.alert(
      //   `${task.id} : id, ${task.dueTime} : Due Time`,
      //   () => {},
      //   `${newTask.name} is added`,
      //   'OK'
      // )
      

      navigator.notification.alert(
        `${dueTime} dueTime`,
        () => {},
        `(${dueTimeDate}) dueTimeDate`,
        'OK'
      )
    },
    handleNotificationTap(notificationAction){
      const notificationId = notificationAction.notification.id;

      const tappedTask = this.tasks.find(task => task.id === Number(notificationId));
      if(tappedTask){
        this.openTaskDialog(tappedTask);
      }
    }
  },
  created(){
    LocalNotifications.addListener('localNotificationActionPerformed', (notificationAction) => {
      this.handleNotificationTap(notificationAction);
    });
  },
  beforeUnmount(){
    LocalNotifications.removeAllListeners();
  }
};
</script>

<style>
  .absolute-bottom-right{
    position: absolute;
    right: 16px;
    bottom: 16px;
  }
</style>