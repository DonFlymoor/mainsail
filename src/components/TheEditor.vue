<template>
    <div>
        <v-dialog
            v-model="show"
            persistent
            fullscreen
            hide-overlay
            :transition="false"
            @close="close"
            @keydown.esc="escClose">
            <panel
                card-class="editor-dialog"
                :icon="isWriteable ? mdiFileDocumentEditOutline : mdiFileDocumentOutline"
                :title="title">
                <template #buttons>
                    <v-btn
                        v-if="restartServiceName === 'klipper'"
                        text
                        tile
                        :href="klipperConfigReference"
                        target="_blank"
                        class="d-none d-md-flex">
                        <v-icon small class="mr-1">{{ mdiHelp }}</v-icon>
                        {{ $t('Editor.ConfigReference') }}
                    </v-btn>
                    <v-btn
                        v-if="isWriteable"
                        text
                        tile
                        :color="restartServiceName === null ? 'primary' : ''"
                        @click="save(null)">
                        <v-icon small class="mr-1">{{ mdiContentSave }}</v-icon>
                        <span class="d-none d-sm-inline">{{ $t('Editor.SaveClose') }}</span>
                    </v-btn>
                    <v-btn
                        v-if="restartServiceNameExists"
                        color="primary"
                        text
                        tile
                        class="d-none d-sm-flex"
                        @click="save(restartServiceName)">
                        <v-icon small class="mr-1">{{ mdiRestart }}</v-icon>
                        {{ $t('Editor.SaveRestart') }}
                    </v-btn>
                    <v-btn icon tile @click="close">
                        <v-icon>{{ mdiCloseThick }}</v-icon>
                    </v-btn>
                </template>
                <v-card-text class="pa-0">
                    <codemirror-async
                        v-if="show"
                        ref="editor"
                        v-model="sourcecode"
                        :name="filename"
                        :file-extension="fileExtension"></codemirror-async>
                </v-card-text>
            </panel>
        </v-dialog>
        <v-snackbar v-model="loaderBool" :timeout="-1" :value="true" fixed right bottom dark>
            <div>
                {{ snackbarHeadline }}
                <br />
                <strong>{{ filename }}</strong>
            </div>
            <template v-if="loaderProgress.total > 0">
                <span class="mr-1">
                    ({{ formatFilesize(loaderProgress.loaded) }}/{{ formatFilesize(loaderProgress.total) }})
                </span>
                {{ Math.round((100 * loaderProgress.loaded) / loaderProgress.total) }} % @ {{ loaderProgress.speed }}/s
                <br />
                <v-progress-linear
                    class="mt-2"
                    :value="(100 * loaderProgress.loaded) / loaderProgress.total"></v-progress-linear>
            </template>
            <template v-else>
                <v-progress-linear class="mt-2" indeterminate></v-progress-linear>
            </template>
            <template #action="{ attrs }">
                <v-btn color="red" text v-bind="attrs" style="min-width: auto" tile @click="cancelDownload">
                    <v-icon class="0">{{ mdiClose }}</v-icon>
                </v-btn>
            </template>
        </v-snackbar>
        <v-dialog v-model="dialogConfirmChange" persistent :width="600">
            <panel
                card-class="editor-confirm-change-dialog"
                :icon="mdiHelpCircle"
                :title="$t('Editor.UnsavedChanges')"
                :margin-bottom="false">
                <template #buttons>
                    <v-btn icon tile @click="dialogConfirmChange = false">
                        <v-icon>{{ mdiCloseThick }}</v-icon>
                    </v-btn>
                </template>
                <v-card-text class="pt-3">
                    <v-row>
                        <v-col>
                            <p class="body-1 mb-2">{{ $t('Editor.UnsavedChangesMessage', { filename: filename }) }}</p>
                            <p class="body-2">{{ $t('Editor.UnsavedChangesSubMessage') }}</p>
                        </v-col>
                    </v-row>
                </v-card-text>
                <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn text @click="discardChanges">
                        {{ $t('Editor.DontSave') }}
                    </v-btn>
                    <v-btn text color="primary" @click="save">
                        {{ $t('Editor.SaveClose') }}
                    </v-btn>
                    <template v-if="restartServiceNameExists">
                        <v-btn text color="primary" @click="save(restartServiceName)">
                            {{ $t('Editor.SaveRestart') }}
                        </v-btn>
                    </template>
                </v-card-actions>
            </panel>
        </v-dialog>
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { capitalize, formatFilesize, windowBeforeUnloadFunction } from '@/plugins/helpers'
import Panel from '@/components/ui/Panel.vue'
import { availableKlipperConfigReferenceTranslations } from '@/store/variables'
import CodemirrorAsync from '@/components/inputs/CodemirrorAsync'
import {
    mdiClose,
    mdiCloseThick,
    mdiContentSave,
    mdiFileDocumentOutline,
    mdiFileDocumentEditOutline,
    mdiHelp,
    mdiHelpCircle,
    mdiRestart,
} from '@mdi/js'
import type Codemirror from '@/components/inputs/Codemirror.vue'

@Component({
    components: { Panel, CodemirrorAsync },
})
export default class TheEditor extends Mixins(BaseMixin) {
    private dialogConfirmChange = false

    formatFilesize = formatFilesize

    /**
     * Icons
     */
    mdiCloseThick = mdiCloseThick
    mdiHelp = mdiHelp
    mdiContentSave = mdiContentSave
    mdiRestart = mdiRestart
    mdiClose = mdiClose
    mdiHelpCircle = mdiHelpCircle
    mdiFileDocumentEditOutline = mdiFileDocumentEditOutline
    mdiFileDocumentOutline = mdiFileDocumentOutline

    private scrollbarOptions = { scrollbars: { autoHide: 'never' } }

    declare $refs: {
        editor: Codemirror
    }

    get changed() {
        return this.$store.state.editor.changed ?? false
    }

    get changedOutput() {
        return this.changed ? '*' : ''
    }

    get show() {
        return this.$store.state.editor.bool ?? false
    }

    get filepath(): string {
        return this.$store.state.editor.filepath ?? ''
    }

    get filename(): string {
        return this.$store.state.editor.filename ?? ''
    }

    get filenameWithoutExtension(): string {
        if (this.filename.lastIndexOf('.')) return this.filename.slice(0, this.filename.lastIndexOf('.'))

        return this.filename
    }

    get fileExtension() {
        if (this.filename.lastIndexOf('.')) return this.filename.slice(this.filename.lastIndexOf('.') + 1)

        return ''
    }

    get fileroot() {
        return this.$store.state.editor.fileroot ?? 'gcodes'
    }

    get permissions(): string {
        return this.$store.state.editor.permissions ?? 'r'
    }

    get isWriteable() {
        return this.permissions.includes('w')
    }

    get sourcecode() {
        return this.$store.state.editor.sourcecode ?? ''
    }

    set sourcecode(newVal) {
        this.$store.dispatch('editor/updateSourcecode', newVal)
    }

    get loaderBool() {
        return this.$store.state.editor.loaderBool ?? false
    }

    get loaderProgress() {
        return this.$store.state.editor.loaderProgress ?? {}
    }

    get snackbarHeadline() {
        let directionUppercase = this.$t('Editor.Downloading')
        if (this.loaderProgress.direction) directionUppercase = capitalize(this.loaderProgress.direction)

        return this.$t(`Editor.${directionUppercase}`)
    }

    get availableServices() {
        return this.$store.state.server.system_info?.available_services ?? []
    }

    get restartServiceName() {
        if (!this.isWriteable) return null
        if (['printing', 'paused'].includes(this.printer_state)) return null

        if (this.availableServices.includes(this.filenameWithoutExtension) && this.fileExtension === 'conf')
            return this.filenameWithoutExtension
        if (this.filename.startsWith('webcam') && ['conf', 'txt'].includes(this.fileExtension)) return 'webcamd'
        if (this.filename.startsWith('mooncord') && this.fileExtension === 'json') return 'mooncord'
        if (this.filename === 'moonraker.conf') return this.moonrakerRestartInstance ?? 'moonraker'

        if (this.fileExtension === 'cfg') return 'klipper'

        return null
    }

    get restartServiceNameExists() {
        if (this.restartServiceName) return true

        return this.availableServices.includes(this.restartServiceName)
    }

    get moonrakerRestartInstance() {
        return this.$store.state.gui.editor.moonrakerRestartInstance
    }

    get confirmUnsavedChanges() {
        return this.$store.state.gui.editor.confirmUnsavedChanges ?? false
    }

    get escToClose() {
        return this.$store.state.gui.editor.escToClose ?? false
    }

    get title() {
        const title = this.filepath ? `${this.filepath}/${this.filename}` : this.filename

        if (!this.isWriteable) return `${title} (${this.$t('Editor.FileReadOnly')})`

        return `${title} ${this.changedOutput}`
    }

    get currentLanguage() {
        return this.$store.state.gui.general.language
    }

    get klipperConfigReference(): string {
        const currentLanguage = this.currentLanguage
        const translations = availableKlipperConfigReferenceTranslations
        let url = 'https://www.klipper3d.org/Config_Reference.html'

        if (translations.includes(currentLanguage)) {
            url = `https://www.klipper3d.org/${currentLanguage}/Config_Reference.html`
        }

        return url
    }

    cancelDownload() {
        this.$store.dispatch('editor/cancelLoad')
    }

    escClose() {
        if (this.escToClose) this.close()
    }

    close() {
        if (this.confirmUnsavedChanges) this.promptUnsavedChanges()
        else this.$store.dispatch('editor/close')
    }

    discardChanges() {
        this.dialogConfirmChange = false
        this.$store.dispatch('editor/close')
    }

    promptUnsavedChanges() {
        if (!this.changed || !this.isWriteable) this.$store.dispatch('editor/close')
        else this.dialogConfirmChange = true
    }

    save(restartServiceName: string | null = null) {
        this.dialogConfirmChange = false

        this.$store.dispatch('editor/saveFile', {
            content: this.sourcecode,
            restartServiceName: restartServiceName,
        })
    }

    @Watch('changed')
    changedChanged(newVal: boolean) {
        if (this.confirmUnsavedChanges) {
            if (newVal) window.addEventListener('beforeunload', windowBeforeUnloadFunction)
            else window.removeEventListener('beforeunload', windowBeforeUnloadFunction)
        }
    }
}
</script>
