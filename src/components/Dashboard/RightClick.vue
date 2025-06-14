<script setup lang="ts">
  import RightClickMenu from '@/components/Core/RightClickMenu'
  import BulkUpdateTrackersDialog from '@/components/Dialogs/BulkUpdateTrackers/BulkUpdateTrackersDialog.vue'
  import CategoryFormDialog from '@/components/Dialogs/CategoryFormDialog.vue'
  import ConfirmDeleteDialog from '@/components/Dialogs/ConfirmDeleteDialog.vue'
  import MoveTorrentDialog from '@/components/Dialogs/MoveTorrentDialog.vue'
  import RenameTorrentDialog from '@/components/Dialogs/RenameTorrentDialog.vue'
  import ShareLimitDialog from '@/components/Dialogs/ShareLimitDialog.vue'
  import SpeedLimitDialog from '@/components/Dialogs/SpeedLimitDialog.vue'
  import TagFormDialog from '@/components/Dialogs/TagFormDialog.vue'
  import { downloadFile } from '@/helpers'
  import { useAppStore, useCategoryStore, useDashboardStore, useDialogStore, useMaindataStore, usePreferenceStore, useTagStore, useTorrentStore } from '@/stores'
  import { RightClickMenuEntryType } from '@/types/vuetorrent'
  import { BlobReader, BlobWriter, ZipWriter } from '@zip.js/zip.js'
  import { computed } from 'vue'
  import { useI18nUtils } from '@/composables'
  import { useRouter } from 'vue-router'
  import { toast } from 'vue3-toastify'

  defineProps<{
    rightClickProperties: { isVisible: boolean; offset: [number, number] }
  }>()

  const { t } = useI18nUtils()
  const router = useRouter()
  const appStore = useAppStore()
  const categoryStore = useCategoryStore()
  const dashboardStore = useDashboardStore()
  const dialogStore = useDialogStore()
  const maindataStore = useMaindataStore()
  const preferenceStore = usePreferenceStore()
  const tagStore = useTagStore()
  const torrentStore = useTorrentStore()

  const isMultiple = computed(() => dashboardStore.selectedTorrents.length > 1)
  const hashes = computed(() => dashboardStore.selectedTorrents)
  const hash = computed(() => hashes.value[0])
  const torrent = computed(() => torrentStore.getTorrentByHash(hash.value))
  const torrents = computed(() => dashboardStore.selectedTorrents.map(torrentStore.getTorrentByHash).filter(torrent => !!torrent))

  async function resumeTorrents() {
    await torrentStore.resumeTorrents(hashes)
  }

  async function forceStartTorrents() {
    await torrentStore.forceStartTorrents(hashes)
  }

  async function pauseTorrents() {
    await torrentStore.pauseTorrents(hashes)
  }

  function deleteTorrents() {
    dialogStore.createDialog(ConfirmDeleteDialog, { hashes: [...dashboardStore.selectedTorrents] })
  }

  function setDownloadPath() {
    dialogStore.createDialog(MoveTorrentDialog, { hashes: [...dashboardStore.selectedTorrents], mode: 'dl' })
  }

  function setSavePath() {
    dialogStore.createDialog(MoveTorrentDialog, { hashes: [...dashboardStore.selectedTorrents], mode: 'save' })
  }

  function renameTorrents() {
    dialogStore.createDialog(RenameTorrentDialog, { hash: dashboardStore.selectedTorrents[0] })
  }

  async function forceRecheck() {
    await torrentStore.recheckTorrents(hashes)
  }

  async function forceReannounce() {
    await torrentStore.reannounceTorrents(hashes)
  }

  async function toggleSuperSeeding() {
    await torrentStore.setSuperSeeding(hashes, !torrent.value?.super_seeding)
  }

  async function toggleSeqDl() {
    await torrentStore.toggleSeqDl(hashes)
  }

  async function toggleFLPiecePrio() {
    await torrentStore.toggleFLPiecePrio(hashes)
  }

  async function toggleAutoTMM() {
    await torrentStore.toggleAutoTmm(hashes, !torrent.value?.auto_tmm)
  }

  function hasTag(tag: string) {
    return torrents.value.every(torrent => torrent && torrent.tags && torrent.tags.includes(tag))
  }

  function openNewTagFormDialog() {
    const selectedHashes = hashes.value
    dialogStore.createDialog(TagFormDialog, { onSubmit: tags => torrentStore.addTorrentTags(selectedHashes, tags) }, maindataStore.forceMaindataSync)
  }

  async function clearAllTags() {
    await torrentStore.removeTorrentTags(hashes.value)
  }

  function openNewCategoryFormDialog() {
    const selectedHashes = hashes.value
    dialogStore.createDialog(CategoryFormDialog, { onSubmit: cat => torrentStore.setTorrentCategory(selectedHashes, cat.name) }, maindataStore.forceMaindataSync)
  }

  async function clearCategory() {
    await torrentStore.setTorrentCategory(hashes.value, '').then(maindataStore.forceMaindataSync)
  }

  async function toggleTag(tag: string) {
    if (hasTag(tag)) await torrentStore.removeTorrentTags(hashes.value, [tag])
    else await torrentStore.addTorrentTags(hashes.value, [tag])
  }

  async function copyValue(valueToCopy: string) {
    try {
      await navigator.clipboard.writeText(valueToCopy)
    } catch (error) {
      toast.error(t('toast.copy.error'))
      return
    }

    toast.success(t('toast.copy.success'))
  }

  function setDownloadLimit() {
    dialogStore.createDialog(SpeedLimitDialog, { hashes: hashes.value, mode: 'download' })
  }

  function setUploadLimit() {
    dialogStore.createDialog(SpeedLimitDialog, { hashes: hashes.value, mode: 'upload' })
  }

  function setShareLimit() {
    dialogStore.createDialog(ShareLimitDialog, { hashes: hashes.value })
  }

  function bulkUpdatetrackers() {
    dialogStore.createDialog(BulkUpdateTrackersDialog, { hashes: hashes.value })
  }

  function waitforme(millisec: number) {
    return new Promise(resolve => {
      setTimeout(() => { resolve('') }, millisec);
    });
  }

  async function openInExternalWebsite(url: string) {
    if (torrents.value) {
      const ts = [...torrents.value]
      let arrs = ts.map((val, _, __) => val.name.split('[')[0]);
      let uniques = [...new Set(arrs)]

      for (let i = 0; i < uniques.length; i++) {
        window.open(url.replace("{0}", uniques[i]), '_blank');
        await waitforme(1000);
      }
    }
  }

  async function openExternalNzbsIn() {
    await openInExternalWebsite('https://nzbs.in/search?category=&query="{0}"');
  }

  async function openExternalOmg() {
    await openInExternalWebsite("https://omgwtfnzbs.org/browse?search={0}");
  }

  async function openExternalSrrDb() {
    await openInExternalWebsite("https://www.srrdb.com/release/details/{0}");
  }

  async function openExternalIntegrity() {
    await openInExternalWebsite("https://integrity.atmegatron.club/?keyword={0}&pageNum=1");
  }

  async function openExternalMyList() {
    await openInExternalWebsite("https://allnzbs.atmegatron.club/?keyword={0}&limit=20&sortBy=TITLE&orientationBy=ASCENDING&location=&pageNum=1");
  }

  async function openExternalShd() {
    await openInExternalWebsite("https://scenehd.org/log.php?search={0}&type=0");
  }

  async function copyWithCategory(multipleSeparator: string = "\n") {
    if (torrents.value) {
      const ts = [...torrents.value]
      let str: string[] = []

      for (let i = 0; i < ts.length; i++) {
        let nm = ts[i].name;
        str.push(ts[i].category + "/" + nm)
      }

      try {
        await navigator.clipboard.writeText(str.join(multipleSeparator))
      } catch (error) {
        toast.error(t('toast.copy.error'))
        return
      }

      toast.success(t('toast.copy.success'))
    }
  }

  async function exportTorrents() {
    const ts = [...torrents.value]
    if (ts.length === 1) {
      const t = ts[0]
      const blob = await torrentStore.exportTorrent(t.hash)
      downloadFile(`${t.name}.torrent`, blob)
      return
    }

    const zipWriter = new ZipWriter(new BlobWriter('application/zip'), { bufferedWrite: true })
    await Promise.all(ts.map(t => torrentStore.exportTorrent(t.hash).then(blob => zipWriter.add(`${t.name}-${t.truncated_hash}.torrent`, new BlobReader(blob)))))
    downloadFile('torrents.zip', await zipWriter.close())
  }

  const menuData = computed<RightClickMenuEntryType[]>(() => [
    {
      text: t('dashboard.right_click.advanced.title'),
      icon: 'mdi-head-cog',
      children: [
        {
          text: t('dashboard.right_click.advanced.download_path'),
          icon: 'mdi-tray-arrow-down',
          action: setDownloadPath
        },
        {
          text: t('dashboard.right_click.advanced.save_path'),
          icon: 'mdi-content-save',
          action: setSavePath
        },
        {
          text: t('dashboard.right_click.advanced.edit_trackers'),
          icon: 'mdi-link-edit',
          action: bulkUpdatetrackers
        },
        {
          text: t('dashboard.right_click.advanced.rename'),
          icon: 'mdi-rename-box',
          hidden: isMultiple.value,
          action: renameTorrents
        },
        {
          text: t('dashboard.right_click.advanced.recheck'),
          icon: 'mdi-playlist-check',
          action: forceRecheck
        },
        {
          text: t('dashboard.right_click.advanced.reannounce'),
          icon: 'mdi-bullhorn',
          action: forceReannounce
        },
        {
          text: t('dashboard.right_click.advanced.super_seeding'),
          icon: torrent.value?.super_seeding ? 'mdi-checkbox-marked' : 'mdi-checkbox-blank-outline',
          action: toggleSuperSeeding
        },
        {
          text: t('dashboard.right_click.advanced.seq_dl'),
          icon: torrent.value?.seq_dl ? 'mdi-checkbox-marked' : 'mdi-checkbox-blank-outline',
          action: toggleSeqDl
        },
        {
          text: t('dashboard.right_click.advanced.f_l_prio'),
          icon: torrent.value?.f_l_piece_prio ? 'mdi-checkbox-marked' : 'mdi-checkbox-blank-outline',
          action: toggleFLPiecePrio
        },
        {
          text: t('dashboard.right_click.advanced.auto_tmm'),
          icon: torrent.value?.auto_tmm ? 'mdi-checkbox-marked' : 'mdi-checkbox-blank-outline',
          action: toggleAutoTMM
        }
      ]
    },
    {
      text: t('dashboard.right_click.priority.title'),
      icon: 'mdi-priority-high',
      hidden: !(preferenceStore.preferences?.queueing_enabled || false),
      children: [
        {
          text: t('dashboard.right_click.priority.top'),
          icon: 'mdi-priority-high',
          action: async () => await torrentStore.setTorrentPriority(hashes.value, 'topPrio')
        },
        {
          text: t('dashboard.right_click.priority.increase'),
          icon: 'mdi-arrow-up',
          action: async () => await torrentStore.setTorrentPriority(hashes.value, 'increasePrio')
        },
        {
          text: t('dashboard.right_click.priority.decrease'),
          icon: 'mdi-arrow-down',
          action: async () => await torrentStore.setTorrentPriority(hashes.value, 'decreasePrio')
        },
        {
          text: t('dashboard.right_click.priority.bottom'),
          icon: 'mdi-priority-low',
          action: async () => await torrentStore.setTorrentPriority(hashes.value, 'bottomPrio')
        }
      ]
    },
    {
      text: t('dashboard.right_click.tags.title'),
      icon: 'mdi-tag',
      disabled: tagStore.tags.length === 0,
      disabledText: t('dashboard.right_click.tags.disabled_title'),
      disabledIcon: 'mdi-tag-off',
      children: tagStore.tags.map(tag => ({
        text: tag,
        icon: hasTag(tag) ? 'mdi-checkbox-marked' : 'mdi-checkbox-blank-outline',
        action: async () => await toggleTag(tag).then(maindataStore.forceMaindataSync)
      })),
      slots: {
        top: [
          {
            text: t('settings.tagsAndCategories.createNewTag'),
            icon: 'mdi-plus',
            action: openNewTagFormDialog
          },
          {
            text: t('settings.tagsAndCategories.deleteUnusedTags'),
            icon: 'mdi-delete',
            action: tagStore.deleteUnusedTags
          },
          {
            text: t('dashboard.right_click.tags.clear_all'),
            icon: 'mdi-playlist-remove',
            hidden: torrent.value?.tags.length === 0,
            action: () => clearAllTags().then(maindataStore.forceMaindataSync)
          }
        ]
      }
    },
    {
      text: t('dashboard.right_click.category.title'),
      icon: 'mdi-label',
      disabled: categoryStore.categories.length === 0,
      disabledText: t('dashboard.right_click.category.disabled_title'),
      disabledIcon: 'mdi-label-off',
      children: categoryStore.categories.map(category => ({
        text: category.name,
        icon: torrent.value?.category === category.name ? 'mdi-label-variant' : undefined,
        action: async () => await torrentStore.setTorrentCategory(hashes.value, category.name).then(maindataStore.forceMaindataSync)
      })),
      slots: {
        top: [
          {
            text: t('settings.tagsAndCategories.createNewCategory'),
            icon: 'mdi-plus',
            action: openNewCategoryFormDialog
          },
          {
            text: t('settings.tagsAndCategories.deleteUnusedCategories'),
            icon: 'mdi-delete',
            action: categoryStore.deleteUnusedCategories
          },
          {
            text: t('dashboard.right_click.category.clear'),
            icon: 'mdi-backspace-reverse',
            hidden: torrent.value?.category.length === 0,
            action: () => clearCategory().then(maindataStore.forceMaindataSync)
          }
        ]
      }
    },
    {
      text: t('dashboard.right_click.speed_limit.title'),
      icon: 'mdi-speedometer-slow',
      children: [
        {
          text: t('dashboard.right_click.speed_limit.download'),
          icon: 'mdi-download',
          action: setDownloadLimit
        },
        {
          text: t('dashboard.right_click.speed_limit.upload'),
          icon: 'mdi-upload',
          action: setUploadLimit
        },
        {
          text: t('dashboard.right_click.speed_limit.share'),
          icon: 'mdi-account-group',
          action: setShareLimit
        }
      ]
    },
    {
      text: t('dashboard.right_click.copy.title'),
      icon: 'mdi-content-copy',
      disabled: !window.isSecureContext,
      hidden: isMultiple.value,
      children: [
        {
          text: t('dashboard.right_click.copy.name'),
          icon: 'mdi-alphabetical-variant',
          action: async () => torrent.value && (await copyValue(torrent.value.name))
        },
        {
          text: t('dashboard.right_click.copy.hash'),
          icon: 'mdi-pound',
          action: async () => await copyValue(hash.value)
        },
        {
          text: t('dashboard.right_click.copy.magnet'),
          icon: 'mdi-magnet',
          action: async () => torrent.value && (await copyValue(torrent.value.magnet))
        },
        {
          text: t('dashboard.right_click.copy.comment'),
          icon: 'mdi-comment-text',
          hidden: !appStore.isFeatureAvailable('5.0.0'),
          disabled: !torrent.value?.comment,
          action: async () => torrent.value && (await copyValue(torrent.value.comment))
        },
        {
          text: t('dashboard.right_click.copy.withCategory'),
          icon: 'mdi-content-copy',
          action: async () => torrent.value && await copyWithCategory(" ")
        }
      ]
    },
    {
      text: t('dashboard.right_click.copy.withCategory'),
      icon: 'mdi-content-copy',
      action: async () => torrent.value && await copyWithCategory(" ")
    },
    {
      text: t('dashboard.right_click.export', dashboardStore.selectedTorrents.length),
      icon: isMultiple.value ? 'mdi-download-multiple' : 'mdi-download',
      action: exportTorrents
    },
    {
      text: t('dashboard.right_click.info'),
      icon: 'mdi-information',
      hidden: isMultiple.value,
      action: () => router.push({ name: 'torrentDetail', params: { hash: hash.value } })
    },
    {
      text: t('dashboard.right_click.open_external'),
      icon: 'mdi-open-in-new',
      children: [
        {
          text: t('dashboard.right_click.open_nzbsin'),
          action: async () => torrent.value && await openExternalNzbsIn()
        },
        {
          text: t('dashboard.right_click.open_omg'),
          action: async () => torrent.value && await openExternalOmg()
        },
        {
          text: t('dashboard.right_click.open_srrdb'),
          action: async () => torrent.value && await openExternalSrrDb()
        },
        {
          text: t('dashboard.right_click.open_integrity'),
          action: async () => torrent.value && await openExternalIntegrity()
        },
        {
          text: t('dashboard.right_click.open_my_list'),
          action: async () => torrent.value && await openExternalMyList()
        },
        {
          text: t('dashboard.right_click.open_shd'),
          action: async () => torrent.value && await openExternalShd()
        }
      ]
    }
  ])
</script>

<template>
  <div :style="`position: absolute; left: ${rightClickProperties.offset[0]}px; top: ${rightClickProperties.offset[1]}px;`">
    <RightClickMenu v-model="rightClickProperties.isVisible" :menu-data="menuData">
      <template v-slot:top>
        <v-list-item>
          <div class="d-flex justify-space-around">
            <v-tooltip location="top">
              <template v-slot:activator="{ props }">
                <v-btn density="compact" variant="plain" icon="mdi-play" v-bind="props" @click="resumeTorrents" />
              </template>
              <span>{{ t('dashboard.right_click.top.resume') }}</span>
            </v-tooltip>

            <v-tooltip location="top">
              <template v-slot:activator="{ props }">
                <v-btn density="compact" variant="plain" icon="mdi-fast-forward" v-bind="props" @click="forceStartTorrents" />
              </template>
              <span>{{ t('dashboard.right_click.top.force_resume') }}</span>
            </v-tooltip>

            <v-tooltip location="top">
              <template v-slot:activator="{ props }">
                <v-btn density="compact" variant="plain" icon="mdi-pause" v-bind="props" @click="pauseTorrents" />
              </template>
              <span>{{ t('dashboard.right_click.top.pause') }}</span>
            </v-tooltip>

            <v-tooltip location="top">
              <template v-slot:activator="{ props }">
                <v-btn color="red" density="compact" variant="plain" icon="mdi-delete-forever" v-bind="props" @click="deleteTorrents" />
              </template>
              <span>{{ t('dashboard.right_click.top.delete') }}</span>
            </v-tooltip>
          </div>
        </v-list-item>
      </template>
    </RightClickMenu>
  </div>
</template>

<style scoped></style>
