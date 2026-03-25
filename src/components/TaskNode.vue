<template>
  <li>
    <div class="task-row" :class="{ child: depth > 0 }">
      <input type="checkbox" :checked="task.done" @change="$emit('toggleDone', task)" />
      <input
        class="task-text"
        :class="{ done: task.done }"
        v-model="task.text"
        @input="$emit('updateTaskText', task, task.text)"
      />
      <span class="priority" :class="task.priority" @click="$emit('cyclePriority', task)">[{{ task.priority }}]</span>
      <button @click="$emit('addSubtask', task.id)">Add subtask</button>
      <button class="danger" @click="$emit('deleteTask', task.id)">Delete</button>
    </div>

    <ul v-if="task.children && task.children.length">
      <TaskNode
        v-for="child in task.children"
        :key="child.id"
        :task="child"
        :depth="depth + 1"
        @toggleDone="$emit('toggleDone', $event)"
        @updateTaskText="$emit('updateTaskText', $event)"
        @cyclePriority="$emit('cyclePriority', $event)"
        @addSubtask="$emit('addSubtask', $event)"
        @deleteTask="$emit('deleteTask', $event)"
      />
    </ul>
  </li>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue'

const props = defineProps({
  task: { type: Object, required: true },
  depth: { type: Number, default: 0 },
})

const emit = defineEmits(['toggleDone', 'updateTaskText', 'cyclePriority', 'addSubtask', 'deleteTask'])
</script>
