<template>
<li class="taskitem" :id="itemId" @mouseover="active" @mouseleave="deactive">
  <div v-if="notPhoneScreen" class="task-actions" :class="{checked: task.checked}">
    <button class="ui button" @click="edit" v-if="!task.checked">
        <i class="write icon"></i>
      </button>
    <button class="ui button" @click="del">
        <i class="trash outline icon"></i>
      </button>
  </div>
  <div class="task-detail" :class="{'checked-state': task.checked, 'over-deadline-state': overDeadline}" v-if="!create && !editable">
    <span class="checkbox-wrapper" :class="{disabled}">
        <input type="checkbox" :checked="task.checked" :id="inputId" :disabled="disabled" @change.once="taskStateChange">
        <label class="disable-checkbox" :for="inputId"></label>
    </span>
    <div class="task-item-content-wrapper">
      <label class="title-label"><a :href="`/task/${task.id}`">{{task.title}}</a></label>
      <div class="task-content">
        <div class="ui image label assignment">
          <img v-if="task.assignee && task.assignee.headimgurl" :src="task.assignee.headimgurl" />
          <span v-else>{{ assignee }}</span> {{ dueDate }}
        </div>
        <assignment-editor :ref="assignmentEditorId" :name="assignmentEditorId" :task="task"></assignment-editor>
      </div>
    </div>
  </div>
  <div class="task-detail" v-if="editable">
    <span class="checkbox-wrapper disabled">
      <input type="checkbox" id="create-task-input" disabled="true">
      <label class="disable-checkbox" for="create-task-input"></label>
    </span>
    <input @keyup.enter="save" type="text" maxlength="500" placeholder="任务名称">
    <div class="task-content">
      <div class="ui image label assignment">
        <img v-if="taskAssignee()" :src="newTask.assignee.headimgurl" />
        <span v-else>未指派</span> {{ taskDueDate() }}
      </div>
      <assignment-editor :ref="assignmentEditorId" :name="assignmentEditorId" :task="task"></assignment-editor>
    </div>
  </div>
  <div v-if="editable" class="task-edit-actions">
    <button class="ui positive button" @click="save">保存</button>
    <button class="ui basic button" @click="dismiss">取消</button>
  </div>
</li>
</template>

<style>
.taskitem {
  width: 100%;
}

.taskitem .checked-state .disable-checkbox {
  background-color: #2cbe4e;
}

.taskitem .checked-state .disable-checkbox:after {
  border-color: lightgrey;
}

.taskitem .checked-state .title-label {
  color: grey;
}

.taskitem .over-deadline-state .title-label {
  color: #ffdf05;
}

.taskitem .task-item-content-wrapper {
  margin-left: 1.2em;
  display: inline-block;
}

.taskitem .task-actions {
  display: inline-block;
  border: 1px solid lightgray;
  border-right: none;
  border-radius: .5em;
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
  line-height: 2.2rem;
  margin-left: 2rem;
  margin-right: -0.8rem;
  margin-top: -0.3rem;
  visibility: hidden;
  vertical-align: top;
}

@media (max-width: 768px) {
  .taskitem .task-actions {
    display: none;
  }
}

.taskitem .task-actions.checked {
  margin-left: 2.2em;
}

.taskitem .task-detail {
  display: inline-block;
  max-width: calc(100% - 109px);
}

@media (max-width: 768px) {
  .taskitem .task-detail {
    max-width: 100%;
  }
}

.taskitem .task-detail .title-label {
  display: inline-block;
  margin-right: .6em;
  max-width: calc(100% - 110px);
  padding-top: 3px;
  line-height: 1.5em;
}

.taskitem .task-detail .title-label>a::after {
  display: block;
  content: attr(title);
  font-weight: bold;
  height: 0;
  overflow: hidden;
  visibility: hidden;
}

.taskitem .task-detail .title-label>a {
  color: black;
}

.taskitem .task-detail .title-label>a:hover {
  color: #4183c4;
}

.taskitem .task-detail .task-item-content-wrapper {
  max-width: calc(100% - 154px);
}

@media (max-width: 1024px) {
  .taskitem .task-detail .task-item-content-wrapper {
    max-width: calc(100% - 60px);
  }
}

.taskitem .task-edit-actions {
  margin-left: 10.8em;
  margin-top: 1em;
  margin-bottom: 2em;
}

.taskitem .task-detail input {
  border: none;
  border-bottom: dashed 1px #aeb3b9;
  min-width: 38em;
  margin-left: .9em;
  margin-right: .9em;
  line-height: 1.8em;
}

.taskitem .task-detail input:focus {
  outline: none;
}

.taskitem.active .task-actions {
  visibility: visible;
}

.taskitem .task-actions .ui.button {
  background: transparent;
  font-size: 1.2em;
  padding: .1em 0;
}

.taskitem .task-actions .ui.button:first-child {
  padding-left: 1.2em;
}

.taskitem .checkbox-wrapper {
  margin-right: 1.2rem;
  float: left;
  width: 20px;
  margin-top: 2px;
}

.taskitem .task-content {
  display: inline-block;
  height: 100%;
  vertical-align: top;
}

.taskitem span>span {
  margin-top: auto;
  margin-bottom: auto;
}

.taskitem span>span:first-child {
  margin-right: .52rem;
}

.ui.image.label.assignment {
  text-align: center;
  cursor: pointer;
  font-weight: normal;
}

.assignment.diasbled {
  cursor: default;
}
</style>

<script>
import {
  DateTime
} from '../utils'
import TaskAssignmentEditor from './TaskAssignmentEditor'
import {
  mapGetters
} from 'vuex'
export default {
  name: 'TaskItem',
  props: ['task', 'disabled', 'create'],
  data() {
    return {
      editable: false,
      switched: false,
      dirty: false,
      newTask: {
        assignee: {},
        deadline: undefined
      },
      overDeadline: false
    }
  },
  mounted() {
    if (!this.disabled && !this.task.checked) {
      this.setupPopups()
    }
    this.$nextTick(() => {
      if (this.task.deadline && (new Date().getTime() - new Date(this.task.deadline).getTime() > 0) && !this.task.checked) this.overDeadline = true
      else this.overDeadline = false
    })
  },
  components: {
    'assignment-editor': TaskAssignmentEditor
  },
  computed: {
    ...mapGetters(['tasks', 'draft']),
    dueDate() {
      if (!this.task.deadline || this.task.deadline == null) return '无限期'
      else return DateTime.DateMonth(this.task.deadline)
    },
    assignee() {
      if (this.task.assignee && this.task.assignee.id) return this.task.assignee.nickname
      else return '未指派'
    },
    itemId() {
      return `task-${this.task.id}`
    },
    inputId() {
      return `task-input-${this.task.id}`
    },
    assignmentEditorId() {
      return `taskitem-assignmenteditor-${this.task.id}`
    },
    notPhoneScreen() {
      return $(document).width() >= 768
    }
  },
  updated() {
    if (!this.task.checked && !this.disabled) {
      this.setupPopups()
    }
    if (this.editable && this.switched) {
      $(this.$el).find('.task-detail > input').val(this.task.title)
      this.switched = false
    }
  },
  watch: {
    disabled(isDisabled) {
      if (isDisabled) {
        $(this.$el).find('.assignment').popup('destroy')
      } else {
        if (!this.task.checked) {
          this.setupPopups()
        }
      }
    }
  },
  methods: {
    active() {
      if (this.disabled || this.editable || this.create) {
        return
      }
      $(this.$el).addClass('active')
    },
    taskDueDate() {
      if (!this.newTask.deadline) return '无限期'
      if (this.newTask.deadline === 'null') return '无限期'
      else return DateTime.DateMonth(this.newTask.deadline)
    },
    taskAssignee() {
      console.log(this.newTask.assignee.id)
      if (this.newTask.assignee.id) return this.newTask.assignee.nickname
      else return undefined
    },
    deactive() {
      $(this.$el).removeClass('active')
    },
    save() {
      this.$emit('editDone')
      let self = this
      let title = $(this.$el).find('.task-detail > input').val()
      if (!title) return
      if (title !== this.task.title) {
        this.task.title = title
        this.dirty = true
      }
      if (this.dirty) {
        if (!this.create) {
          this.$store.dispatch('updateTask', this.task).then(task => {
            self.dirty = false
            if (self.$route.name === 'DraftDiscussion') return self.$store.dispatch('getLatestDraftPost', task.draft_id)
            if (self.$route.name === 'TaskDiscussion') return self.$store.dispatch('getLatestTaskPost', task.id)
          }).catch(() => {
            self.dirty = false
          })
        } else {
          this.$store.dispatch('createTask', {
            draftId: this.draft.id,
            task: this.task
          }).then(task => {
            self.dirty = false
            return self.$store.dispatch('getLatestDraftPost', task.draft_id)
          }).catch(() => {
            self.dirty = false
          })
        }
      }
      this.editable = false
      this.switched = true
      this.newTask = {
        assignee: {},
        deadline: undefined
      }
    },
    dismiss() {
      let self = this
      this.$emit('editDone')
      this.editable = false
      this.switched = true
      const task = this.tasks.find(t => {
        return t.id === self.task.id
      })
      this.task.assignee = task.assignee
      this.task.title = task.title
      this.task.deadline = task.deadline
      this.newTask = {
        assignee: {},
        deadline: undefined
      }
    },
    edit() {
      this.editable = true
      this.switched = true
      this.newTask = JSON.parse(JSON.stringify(this.task))
      if (!this.task.assignee) this.newTask.assignee = {}
      if (!this.task.deadline) this.newTask.deadline = undefined
      $(this.$el).removeClass('active')
    },
    setEditable(editable) {
      if (this.editable !== editable) {
        this.switched = true
        this.editable = editable
      }
    },
    taskStateChange() {
      let isCheck = $(this.$el).find(`#${this.inputId}`).is(':checked')
      if (isCheck !== this.task.checked) {
        this.task.checked = isCheck
        this.$store.dispatch('updateTask', this.task).then((task) => {
          if (this.$route.name === 'DraftDiscussion') return this.$store.dispatch('getLatestDraftPost', task.draft_id)
          if (this.$route.name === 'TaskDiscussion') return this.$store.dispatch('getLatestTaskPost', task.id)
        })
      }
    },
    setupPopups() {
      let self = this
      $(this.$el).find('.assignment').popup({
        lastResort: 'right center',
        position: 'right center',
        hoverable: true,
        on: 'click',
        onShow() {
          if (self.task.assignee) {
            self.$refs[self.assignmentEditorId].setSelection(self.task.assignee.id)
          }
          return true
        },
        onHide() {
          function updateAssignment(task) {
            let dirty = false
            let assignment = self.$refs[self.assignmentEditorId].getData()
            if (assignment.deadline === 'null') {
              // deadline cleared
              if (task.deadline) {
                // origin task has deadline, this will not be null if it's from server
                task.deadline = 'null'
                dirty = true
              }
            } else if (assignment.deadline && assignment.deadline !== 'null') {
              // deadline has value, update the task when task has no deadline or they are not the same
              if (!task.deadline || DateTime.DateMonth(task.deadline) !== DateTime.DateMonth(assignment.deadline)) {
                task.deadline = assignment.deadline.format()
                dirty = true
              }
            } // else leave the deadline unchanged

            // console.log(task.assignee)
            // console.log(assignment.assignee)
            if (assignment.assignee.id === 'null') {
              // assignee cleared
              if (task.assignee) {
                // origin task has assignee, this will not be null if it's from server
                task.assignee = assignment.assignee
                dirty = true
              }
            } else if (assignment.assignee.id && assignment.assignee.id !== 'null') {
              // assignee has value, update the task when task has no assignee or they not the same
              if (!task.assignee || task.assignee.id !== assignment.assignee.id) {
                task.assignee = assignment.assignee
                dirty = true
              }
            } // else leave the assignee unchanged
            return dirty
          }
          if (!self.editable) {
            if (updateAssignment(self.task)) {
              self.$store.dispatch('updateTask', self.task).then(task => {
                if (self.$route.name === 'DraftDiscussion') return self.$store.dispatch('getLatestDraftPost', task.draft_id)
                if (self.$route.name === 'TaskDiscussion') return self.$store.dispatch('getLatestTaskPost', task.id)
              })
              self.dirty = true
            }
          } else {
            if (updateAssignment(self.newTask)) {
              if (self.newTask.deadline === 'null') {
                self.task.deadline = null
              } else if (self.newTask.deadline) {
                self.task.deadline = self.newTask.deadline
              } else {
                self.newTask.deadline = undefined
              }
              if (self.newTask.assignee.id) self.task.assignee = self.newTask.assignee
              self.dirty = true
            }
          }
          return true
        }
      })
    },
    del() {
      let self = this
      $('#task-deletion-modal').modal({
        closable: true,
        onApprove: function () {
          self.$store.dispatch('delTask', self.task).then(() => {
            self.$store.dispatch('getDraftPosts', { draftId: self.draft.id, pageNumber: 1 })
          })
        }
      }).modal('show')
    }
  }
}
</script>
