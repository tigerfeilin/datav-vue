<template>
  <div class="my-project">
    <div class="project-main">
      <div class="project-manage">
        <div class="manage-title">
          <div class="my-project project-group">
            <span style="margin-left: 2px;">我的分组</span>
            <i class="v-icon-plus btn-add-icon" @click="adding = true"></i>
          </div>
          <div
            class="my-project project-all"
            :class="{ 'project-checked-color': selectedGroupId === group.id }"
            @click="toggleProject(group.id)"
          >
            <span>{{ group.name }}</span>
            <span class="project-num">{{ group.children.length }}</span>
          </div>
          <div v-if="adding" class="new-group">
            <input
              v-focus
              class="edit-input"
              @blur="onAddInputBlur"
              @keyup.enter="addGroup"
            >
          </div>
        </div>

        <div class="manage-main" :class="{ draging: draging }">
          <div
            class="main-project"
            :class="{ 'project-checked-color': selectedGroupId === ungroup.id }"
            @click="toggleProject(ungroup.id)"
            @dragover.prevent
            @dragenter="onDragEnter"
            @dragleave="onDragLeave"
            @drop="onDrop($event, ungroup)"
          >
            <span class="project-name project-ungrouped">{{ ungroup.name }}</span>
            <span class="project-num">{{ ungroup.children.length }}</span>
          </div>

          <div
            v-for="g in groups"
            :key="g.id"
            class="main-project group-project"
            :class="{ 'project-checked-color': selectedGroupId === g.id }"
            @click="toggleProject(g.id)"
            @dragover.prevent
            @dragenter="onDragEnter"
            @dragleave="onDragLeave"
            @drop="onDrop($event, g)"
          >
            <template v-if="g.editing">
              <input
                v-focus
                :default-value="g.name"
                class="edit-input"
                @blur="onEditInputBlur($event, g)"
                @keyup.enter="editGroup($event, g)"
              >
            </template>
            <template v-else>
              <span class="project-name">{{ g.name }}</span>
              <span class="project-num">{{ g.children.length }}</span>
              <span class="group-btns">
                <i class="v-icon-edit" @click="g.editing = true"></i>
                <i class="v-icon-delete" @click="confirmDeleteGroup(g)"></i>
              </span>
            </template>
          </div>
        </div>
      </div>
      <div class="project-screen-list">
        <g-loading :spinning="loading">
          <project-list :group="selectedGroup" />
        </g-loading>
      </div>
    </div>
  </div>
</template>

<script lang='ts'>
import { defineComponent, ref, computed, provide, onMounted } from 'vue'
import { ProjectGroup } from '@/domains/project'
import { ProjectModule } from '@/store/modules/project'
import { MessageBoxUtil, MessageUtil } from '@/utils/message-util'
import { addClass, removeClass } from '@/utils/dom'
import ProjectList from './project-list.vue'

export default defineComponent({
  name: 'MyProject',
  components: {
    ProjectList,
  },
  setup() {
    const {
      getProjects, moveProject,
      createProjectGroup, deleteProjectGroup, updateProjectGroupName,
    } = ProjectModule

    const selectedGroupId = ref(-1)
    const loading = ref(true)
    const adding = ref(false)
    const draging = ref(false)

    const group = computed(() => ProjectModule.group)
    const ungroup = computed(() => ProjectModule.ungroup)
    const groups = computed(() => ProjectModule.groups)

    const selectedGroup = computed(() => {
      if (selectedGroupId.value === -1) {
        return group.value
      }

      if (selectedGroupId.value === 0) {
        return ungroup.value
      }

      return groups.value.find(g => g.id === selectedGroupId.value)
    })

    const toggleProject = (id: number) => {
      selectedGroupId.value = id
    }

    const onAddInputBlur = (e: any) => {
      if (!adding.value) {
        return
      }

      const name = (e.target.value || '').trim()
      if (!name) {
        adding.value = false
      }
    }

    const addGroup = async (e: any) => {
      if (!adding.value) {
        return
      }

      const name = (e.target.value || '').trim()
      if (name) {
        try {
          await createProjectGroup(name)
          adding.value = false
        } catch (error) {
          MessageUtil.error(MessageUtil.format(error))
        }
      } else {
        adding.value = false
      }
    }

    const onEditInputBlur = (e: any, group: any) => {
      if (!group.editing) {
        return
      }

      const newName = (e.target.value || '').trim()
      if (!newName || group.name === newName) {
        group.editing = false
      }
    }

    const editGroup = async (e: any, group: any) => {
      if (!group.editing) {
        return
      }

      const newName = (e.target.value || '').trim()
      if (newName && group.name !== newName) {
        try {
          await updateProjectGroupName({ id: group.id, newName })
          group.name = newName
          group.editing = false
        } catch (error) {
          MessageUtil.error(MessageUtil.format(error))
        }
      } else {
        group.editing = false
      }
    }

    const confirmDeleteGroup = (group: ProjectGroup) => {
      MessageBoxUtil.confirmAsync(
        `<b>${group.name}</b> 删除后无法恢复，该分组中的可视化应用将全部移动到未分组，确认删除？`,
        () => {
          return deleteProjectGroup(group.id)
        },
        {
          success: () => {
            toggleProject(ungroup.value.id)
          },
        },
      )
    }

    provide('dragStart', () => {
      draging.value = true
    })
    provide('dragEnd', () => {
      draging.value = false
    })

    const onDragEnter = (event: any) => {
      addClass(event.target, 'drag-enter')
    }

    const onDragLeave = (event: any) => {
      removeClass(event.target, 'drag-enter')
    }

    const onDrop = (event: any, toGroup: ProjectGroup) => {
      event.preventDefault()
      removeClass(event.target, 'drag-enter')

      const str = event.dataTransfer.getData('text')
      if (str) {
        const [pid, fromId] = str.split(',').map((m: string) => parseInt(m))
        if (fromId !== toGroup.id) {
          moveProject({ pid, fromId, toId: toGroup.id })
        }
      }
    }

    onMounted(() => {
      getProjects().finally(() => {
        loading.value = false
      })
    })

    return {
      loading,
      group,
      ungroup,
      groups,
      selectedGroup,
      selectedGroupId,
      adding,
      toggleProject,
      onAddInputBlur,
      addGroup,
      onEditInputBlur,
      editGroup,
      confirmDeleteGroup,
      draging,
      onDragEnter,
      onDragLeave,
      onDrop,
    }
  },
})
</script>

<style lang="scss" scoped>
@import '@/styles/themes/var';
@import '@/styles/mixins/util';
@import '@/styles/mixins/function';

.my-project {
  position: relative;
  user-select: none;

  .project-main {
    display: flex;
    flex: 1;
  }

  .project-manage {
    min-width: 240px;
    max-width: 240px;
    color: #fff;
    position: sticky;
    top: 70px;
    font-size: 14px;
    overflow-y: auto;
    height: calc(100vh - 150px);

    .manage-main {
      display: flex;
      flex-direction: column;
      font-size: 12px;

      &.draging {
        background: rgba(36, 127, 255, 0.3);
      }
    }

    .my-project {
      line-height: 36px;
      height: 36px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding-right: 30px;
    }

    .manage-title {
      .project-group {
        padding-left: 24px;
        height: 56px;
        border-bottom: 1px solid $nav-border-color;
      }

      .btn-add-icon {
        cursor: pointer;

        &:hover {
          color: $color-primary;
        }
      }
    }

    .project-ungrouped {
      color: $font-color;
    }

    .project-all {
      padding-left: 50px;
      transition: color 0.2s;
      cursor: pointer;

      &:hover {
        color: $color-primary;
      }
    }

    .project-checked-color {
      background-image: url(com-cdn('datav/nav-menu-img.png'));
      background-repeat: round;

      &:hover {
        color: #fff !important;
      }

      .project-name {
        color: #fff !important;
      }
    }

    .project-num {
      user-select: none;
      color: $font-color;
    }

    .group-btns {
      display: none;
      color: $color-primary;

      i + i {
        margin-left: 10px;
      }
    }

    .main-project {
      position: relative;
      height: 36px;
      padding: 0 30px 0 50px;
      display: flex;
      font-size: 12px;
      align-items: center;
      justify-content: space-between;
      cursor: pointer;
      transition: color 0.2s;

      .project-name {
        width: 100px;
        font-size: 14px;
        text-overflow: ellipsis;
        overflow: hidden;
        white-space: nowrap;
        pointer-events: none;
      }

      &:hover {
        .project-name {
          color: $color-primary;
        }
      }

      &.drag-enter {
        background: $background-color;
      }
    }

    .group-project {
      &:hover {
        .project-num {
          display: none;
        }

        .group-btns {
          display: inline-block;
        }
      }
    }
  }

  .new-group {
    padding: 5px 30px 5px 50px;
  }

  .edit-input {
    @include utils-ellipsis;

    background: $background-color-dark;
    color: #fff;
    padding: 0 10px;
    line-height: 30px;
    width: 100%;
    height: 30px;
    border: $border-primary;
    transition: 0.2s;
    box-shadow: $shadow;
  }

  .project-screen-list {
    padding-left: 20px;
    padding-right: 50px;
    width: 100%;
  }
}
</style>
