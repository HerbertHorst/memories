<template>
  <Modal ref="modal" @close="cleanup" v-if="show" size="large">
    <template #title>
      {{ t('memories', 'Add person') }}
    </template>

    <div class="manual-face-add">
      <!-- Step 1: pick a file -->
      <div v-if="!fileId" class="picker-step">
        <NcNoteCard v-if="loadError" type="error">{{ loadError }}</NcNoteCard>
        <p>{{ t('memories', 'Choose a photo containing the person you want to tag.') }}</p>
        <NcButton type="primary" @click="pickFile">
          {{ t('memories', 'Choose photo') }}
        </NcButton>
      </div>

      <!-- Step 2: draw rectangles -->
      <div v-else class="draw-step">
        <!-- Suggestions of already-known person names for the name fields below -->
        <datalist id="memories-manual-face-names">
          <option v-for="n in knownNames" :key="n" :value="n" />
        </datalist>

        <NcNoteCard v-if="loadError" type="error">
          {{ loadError }}
          <NcButton type="tertiary" @click="loadFaces">{{ t('memories', 'Try again') }}</NcButton>
        </NcNoteCard>

        <NcNoteCard v-if="!dimensionsKnown" type="warning">
          {{
            t(
              'memories',
              'The original size of this photo is not known yet, so a marking would end up in the wrong place. Faces can be marked on it once the photo has been indexed.',
            )
          }}
        </NcNoteCard>

        <NcNoteCard v-if="legacy" type="info">
          {{
            t(
              'memories',
              'Face Recognition on the server does not report the state of the faces. Marking without a name, searching an area and moving a whole group become available once it is updated.',
            )
          }}
        </NcNoteCard>

        <NcNoteCard v-if="navigationError" type="info">
          {{
            t(
              'memories',
              'Zooming with two fingers is not available here. Drawing, the buttons and the mouse wheel still work.',
            )
          }}
        </NcNoteCard>

        <p class="hint">{{ hint }}</p>

        <div class="stage-wrap">
          <FaceMarkingStage
            :src="imageSrc"
            :faces="stageFaces"
            :regions="stageRegions"
            :rect="rect"
            :drawing-enabled="canDraw"
            @update:rect="onRect"
            @select="selectFace"
            @navigation-error="navigationError = true"
            @image-error="loadError = t('memories', 'The photo could not be loaded.')"
          />
          <transition name="saved-notice">
            <div v-if="notice" class="saved-notice" role="status">{{ notice }}</div>
          </transition>
        </div>

        <div class="legend">
          <span><i class="swatch origin-auto" />{{ t('memories', 'Found automatically') }}</span>
          <span><i class="swatch origin-manual" />✎ {{ t('memories', 'Marked by hand') }}</span>
          <template v-if="!legacy">
            <span><i class="swatch line-participating" />{{ t('memories', 'Used for recognition') }}</span>
            <span><i class="swatch line-pending" />{{ t('memories', 'Waiting') }}</span>
            <span><i class="swatch line-excluded" />{{ t('memories', 'Not used') }}</span>
          </template>
        </div>

        <div v-if="regions && regions.length" class="regions">
          <div class="section-title">{{ t('memories', 'Searched areas') }}</div>
          <ul>
            <li v-for="region in regions" :key="region.id">{{ regionText(region) }}</li>
          </ul>
        </div>

        <!-- A new marking -->
        <div v-if="rect && !selectedFace" class="fields">
          <NcTextField
            ref="nameField"
            class="field"
            :value.sync="rawInput"
            :label="t('memories', 'Name')"
            :label-visible="false"
            :placeholder="t('memories', 'Name')"
            list="memories-manual-face-names"
            @keypress.enter="saveNamed()"
          />
          <NcNoteCard v-if="saveError" type="error">{{ saveError }}</NcNoteCard>
          <p class="scope">{{ markingScope }}</p>
          <p v-if="canSearchRegion" class="scope">{{ t('memories', 'Or search the area:') }} {{ regionScope }}</p>
        </div>

        <!-- A face that is there -->
        <div v-if="selectedFace" class="fields">
          <div class="face-title">{{ nameOf(selectedFace) }}</div>
          <p class="state">{{ originText(selectedFace) }}</p>
          <p v-if="selectedParticipation" class="state">{{ selectedParticipation }}</p>
          <p v-for="(hintText, i) in selectedHints" :key="i" class="state hint-text">{{ hintText }}</p>

          <template v-if="canReassign">
            <NcTextField
              ref="editField"
              class="field"
              :value.sync="editName"
              :label="t('memories', 'Name')"
              :label-visible="false"
              :placeholder="t('memories', 'Name')"
              list="memories-manual-face-names"
              @keypress.enter="saveEdit()"
            />
            <p v-if="groupSize !== null" class="group">
              {{
                n('memories', 'Its group has {count} face.', 'Its group has {count} faces.', groupSize, {
                  count: groupSize,
                })
              }}
              <a :href="groupHref" target="_blank" rel="noopener noreferrer">{{
                t('memories', 'Show the photos of this group')
              }}</a>
            </p>
            <NcCheckboxRadioSwitch v-if="canMoveGroup" :checked.sync="wholeGroup">
              {{ t('memories', 'Move the whole group') }}
            </NcCheckboxRadioSwitch>
            <NcNoteCard v-if="saveError" type="error">{{ saveError }}</NcNoteCard>
            <p v-if="editTarget" class="scope">{{ reassignScope }}</p>
          </template>
          <p v-else class="hint">
            {{
              t(
                'memories',
                'This face is not in a group yet. It can be named once the face recognition has placed it in one, on its next run.',
              )
            }}
          </p>
        </div>
      </div>
    </div>

    <template #buttons>
      <NcButton v-if="fileId && !rect && !selectedFace" @click="resetFile">
        {{ t('memories', 'Choose different photo') }}
      </NcButton>

      <template v-if="rect && !selectedFace">
        <NcButton v-if="canSearchRegion" :disabled="saving" @click="saveRegion">
          {{ t('memories', 'Search area for faces') }}
        </NcButton>
        <NcButton v-if="canSaveUnnamed" :disabled="saving" @click="saveUnnamed">
          {{ t('memories', 'Save without name') }}
        </NcButton>
        <NcButton class="button" type="primary" :disabled="!canSaveNamed" @click="saveNamed">
          {{ t('memories', 'Save') }}
        </NcButton>
      </template>

      <template v-if="selectedFace">
        <NcButton @click="cancelEdit">
          {{ t('memories', 'Cancel') }}
        </NcButton>
        <NcButton v-if="canReassign" class="button" type="primary" :disabled="!canSaveEdit" @click="saveEdit">
          {{ t('memories', 'Save') }}
        </NcButton>
      </template>
    </template>
  </Modal>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import axios from '@nextcloud/axios';
import { getFilePickerBuilder } from '@nextcloud/dialogs';

import NcButton from '@nextcloud/vue/dist/Components/NcButton.js';
import NcNoteCard from '@nextcloud/vue/dist/Components/NcNoteCard.js';
const NcTextField = () => import('@nextcloud/vue/dist/Components/NcTextField.js');
const NcCheckboxRadioSwitch = () => import('@nextcloud/vue/dist/Components/NcCheckboxRadioSwitch.js');

import Modal from './Modal.vue';
import ModalMixin from './ModalMixin';
import FaceMarkingStage from './FaceMarkingStage.vue';

import { API } from '@services/API';
import { translate as t } from '@services/l10n';
import * as utils from '@services/utils';
import {
  faceRecognitionAddManualFace,
  faceRecognitionAddManualRegion,
  faceRecognitionAssignCluster,
  faceRecognitionGetFacesForFile,
  getFaceList,
  type IFaceLimits,
  type IFaceRectForFile,
  type IManualRegion,
} from '@services/dav/face';

import {
  hintsOf,
  markingScope,
  nameOf,
  originText,
  participationText,
  reassignScope,
  regionScope,
  regionText,
  stageFaceOf,
  stageRegionOf,
  type Rect,
  type StageFace,
  type StageRegion,
} from './faceMarking';

/** How long a saved marking is announced over the photo */
const NOTICE_MS = 4000;

/** How often, and how far apart, a name field that is still loading is looked for */
const FOCUS_TRIES = 20;
const FOCUS_RETRY_MS = 50;

/** The answer of the server that came with a failed request, if there was one. */
function responseOf(e: unknown): { status?: number; data?: { error?: unknown } } | null {
  if (typeof e !== 'object' || e === null || !('response' in e)) return null;
  const response = (e as { response?: unknown }).response;
  return typeof response === 'object' && response !== null ? response : null;
}

export default defineComponent({
  name: 'FaceManualAddModal',
  components: { NcButton, NcNoteCard, NcTextField, NcCheckboxRadioSwitch, Modal, FaceMarkingStage },

  mixins: [ModalMixin],

  emits: {
    /** Faces were added or moved while the dialog was open; sent when it closes */
    added: () => true,
  },

  data: () => ({
    fileId: 0,
    imageSrc: '',
    imageNatW: 0,
    imageNatH: 0,
    faces: [] as IFaceRectForFile[],
    regions: null as IManualRegion[] | null,
    limits: null as IFaceLimits | null,
    legacy: false,
    loadError: '',
    rect: null as Rect | null,
    rawInput: '',
    saving: false,
    saveError: '',
    selectedFaceId: null as number | null,
    editName: '',
    wholeGroup: false,
    knownNames: [] as string[],
    navigationError: false,
    /** What was just saved, shown over the photo for a moment */
    notice: '',
    noticeTimer: 0,
    /**
     * Whether anything was saved. The sidebar and the timeline are told when
     * the dialog closes, not on each save: the sidebar reloads by dropping
     * its content, and this dialog with it.
     */
    changed: false,
  }),

  computed: {
    /** The size of the original photo, which every marking is measured against. */
    dimensionsKnown(): boolean {
      return this.imageNatW > 0 && this.imageNatH > 0;
    },

    canDraw(): boolean {
      return this.dimensionsKnown && !this.saving;
    },

    name(): string {
      return this.rawInput.trim();
    },

    canSaveNamed(): boolean {
      return !!this.rect && !!this.name && !this.saving && this.dimensionsKnown;
    },

    canSaveUnnamed(): boolean {
      return !this.legacy;
    },

    canSearchRegion(): boolean {
      return !this.legacy && this.regions !== null;
    },

    hint(): string {
      if (!this.dimensionsKnown) return '';
      if (this.selectedFace) {
        return t('memories', 'Draw a new rectangle to mark another face.');
      }
      if (this.rect) {
        return t('memories', 'Enter a name, or save without one. You can redraw by dragging again.');
      }
      return t(
        'memories',
        'Drag on the photo to mark a face, or an area with several faces to search again. Zoom with the mouse wheel or two fingers, and move with the middle mouse button or two fingers. Click a face to see its state or rename it.',
      );
    },

    stageFaces(): StageFace[] {
      if (!this.dimensionsKnown) return [];
      return this.faces.map((face) =>
        stageFaceOf(face, this.imageNatW, this.imageNatH, this.limits, face.id === this.selectedFaceId),
      );
    },

    stageRegions(): StageRegion[] {
      if (!this.dimensionsKnown || !this.regions) return [];
      return this.regions.map((region) => stageRegionOf(region, this.imageNatW, this.imageNatH));
    },

    markingScope(): string {
      return markingScope(this.name, this.knownNames);
    },

    regionScope(): string {
      return regionScope();
    },

    selectedFace(): IFaceRectForFile | null {
      return this.faces.find((face) => face.id === this.selectedFaceId) ?? null;
    },

    selectedParticipation(): string {
      return this.selectedFace ? participationText(this.selectedFace, this.limits) : '';
    },

    selectedHints(): string[] {
      return this.selectedFace ? hintsOf(this.selectedFace) : [];
    },

    /** A face can only be moved to a person through its group. */
    canReassign(): boolean {
      return this.selectedFace?.cluster !== null && this.selectedFace?.cluster !== undefined;
    },

    groupSize(): number | null {
      return this.selectedFace?.clusterSize ?? null;
    },

    /** Moving the whole group is offered when there is more in it than this face. */
    canMoveGroup(): boolean {
      return !this.legacy && this.groupSize !== null && this.groupSize > 1;
    },

    groupHref(): string {
      const cluster = this.selectedFace?.cluster;
      if (cluster === null || cluster === undefined) return '';
      return this.$router.resolve({ name: 'facerecognition', params: { user: utils.uid ?? '', name: String(cluster) } })
        .href;
    },

    editTarget(): string {
      return this.editName.trim();
    },

    reassignScope(): string {
      return reassignScope(this.editTarget, this.wholeGroup && this.canMoveGroup, this.groupSize);
    },

    canSaveEdit(): boolean {
      return this.canReassign && !!this.editTarget && !this.saving;
    },
  },

  methods: {
    nameOf,
    originText,
    regionText,

    open() {
      this.resetAll();
      this.changed = false;
      this.show = true;
      this.loadKnownNames();
    },

    async openForFile(info: { fileid: number; etag?: string; w?: number; h?: number }) {
      this.resetAll();
      this.changed = false;
      this.show = true;
      this.loadKnownNames();
      if (!info?.fileid) return;

      this.fileId = info.fileid;
      // Only the size of the original is any good: a marking is measured
      // against it, and the preview is scaled down.
      this.imageNatW = info.w ?? 0;
      this.imageNatH = info.h ?? 0;
      this.imageSrc = this.previewOf(this.fileId, info.etag);
      await this.loadFaces();
    },

    cleanup() {
      this.show = false;
      this.resetAll();
      if (this.changed) {
        this.changed = false;
        this.$emit('added');
      }
    },

    /** Shows what was just saved over the photo, for a few seconds. */
    showNotice(text: string) {
      window.clearTimeout(this.noticeTimer);
      this.notice = text;
      this.noticeTimer = window.setTimeout(() => (this.notice = ''), NOTICE_MS);
    },

    /**
     * Puts the cursor in a name field without scrolling to it: the field
     * appears below the photo while the user is still looking at it. Only
     * with a mouse; on a touch screen the keyboard would cover the photo.
     */
    focusField(ref: 'nameField' | 'editField') {
      if (!window.matchMedia?.('(pointer: fine)').matches) return;
      // NcTextField is loaded on first use, so the field may take a moment to
      // be there; it is looked for a few times before giving up.
      let tries = FOCUS_TRIES;
      const attempt = () => {
        const field = this.$refs[ref] as { $el?: Element } | undefined;
        const input = field?.$el?.querySelector?.('input');
        if (input) {
          input.focus({ preventScroll: true });
        } else if (--tries > 0) {
          window.setTimeout(attempt, FOCUS_RETRY_MS);
        }
      };
      this.$nextTick(attempt);
    },

    previewOf(fileId: number, etag?: string): string {
      return API.Q(API.IMAGE_PREVIEW(fileId), { c: etag, x: 2048, y: 2048, a: '1' });
    },

    /**
     * Load the faces of the photo, with their state, and the regions queued
     * on it. A failure is shown here, and leaves the rest of the app alone.
     */
    async loadFaces(): Promise<void> {
      const fileId = this.fileId;
      if (!fileId) return;
      this.loadError = '';
      try {
        const result = await faceRecognitionGetFacesForFile(fileId);
        // The dialog moved on to another photo, or was closed, meanwhile.
        if (fileId !== this.fileId) return;
        this.faces = result.faces;
        this.regions = result.regions;
        this.limits = result.limits;
        this.legacy = result.legacy;
        if (this.selectedFaceId !== null && !this.selectedFace) {
          this.selectedFaceId = null;
        }
      } catch (e) {
        console.error(e);
        if (fileId !== this.fileId) return;
        this.loadError = this.errorText(e, t('memories', 'The faces of this photo could not be loaded.'));
      }
    },

    /**
     * Load the names of already-known persons so the name fields can offer
     * autocompletion — mirrors how naming an unknown cluster suggests existing names.
     * Failure is non-fatal: it just means no suggestions are shown.
     */
    async loadKnownNames(): Promise<void> {
      try {
        const faces = await getFaceList('facerecognition');
        const names = faces
          .map((f) => f.name)
          // Keep only real names; unnamed clusters expose a numeric id as their name.
          .filter((n): n is string => !!n && Number.isNaN(Number(n)));
        this.knownNames = Array.from(new Set(names)).sort((a, b) => a.localeCompare(b));
      } catch (e) {
        console.error(e);
        this.knownNames = [];
      }
    },

    resetAll() {
      this.resetFile();
      this.knownNames = [];
      this.navigationError = false;
    },

    resetFile() {
      this.fileId = 0;
      this.imageSrc = '';
      this.imageNatW = 0;
      this.imageNatH = 0;
      this.faces = [];
      this.regions = null;
      this.limits = null;
      this.legacy = false;
      this.loadError = '';
      this.rect = null;
      this.rawInput = '';
      this.saving = false;
      this.saveError = '';
      this.selectedFaceId = null;
      this.editName = '';
      this.wholeGroup = false;
      window.clearTimeout(this.noticeTimer);
      this.notice = '';
    },

    async pickFile(): Promise<void> {
      const picker = getFilePickerBuilder(t('memories', 'Choose photo'))
        .setMultiSelect(false)
        .addMimeTypeFilter('image/jpeg')
        .addMimeTypeFilter('image/png')
        .addMimeTypeFilter('image/webp')
        .addMimeTypeFilter('image/heic')
        .addMimeTypeFilter('image/heif')
        .setType(1)
        .allowDirectories(false)
        .build();

      let path: string;
      try {
        path = (await picker.pick()) as string;
      } catch (e) {
        return; // user cancelled
      }
      if (!path) return;

      await this.loadPhotoByPath(path);
    },

    async loadPhotoByPath(path: string): Promise<void> {
      try {
        // Memories' IMAGE_INFO requires a fileid, so the file is looked up via WebDAV.
        const props = await this.webdavFileInfo(path);
        this.fileId = props.fileid;
        this.imageNatW = props.w;
        this.imageNatH = props.h;
        this.imageSrc = this.previewOf(this.fileId, props.etag);
      } catch (e) {
        console.error(e);
        this.resetFile();
        this.loadError = t('memories', 'Failed to load the selected photo.');
        return;
      }
      await this.loadFaces();
    },

    async webdavFileInfo(path: string): Promise<{ fileid: number; etag: string; w: number; h: number }> {
      const url = `/remote.php/dav/files/${encodeURIComponent((window as any).OC?.getCurrentUser?.().uid || '')}${path
        .split('/')
        .map(encodeURIComponent)
        .join('/')}`;
      const body = `<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:oc="http://owncloud.org/ns" xmlns:nc="http://nextcloud.org/ns">
  <d:prop>
    <oc:fileid/>
    <d:getetag/>
    <nc:metadata-photos-size/>
  </d:prop>
</d:propfind>`;
      const res = await axios.request({
        method: 'PROPFIND',
        url,
        data: body,
        headers: { Depth: '0', 'Content-Type': 'application/xml' },
      });
      const text = typeof res.data === 'string' ? res.data : new XMLSerializer().serializeToString(res.data);
      const doc = new DOMParser().parseFromString(text, 'application/xml');
      const fileid = parseInt(
        doc.getElementsByTagNameNS('http://owncloud.org/ns', 'fileid')[0]?.textContent ?? '0',
        10,
      );
      const etag = (doc.getElementsByTagNameNS('DAV:', 'getetag')[0]?.textContent ?? '').replace(/"/g, '');
      const sizeEl =
        doc.getElementsByTagNameNS('http://nextcloud.org/ns', 'metadata-photos-size')[0]?.textContent ?? '';
      // Without the size of the original the photo is shown, but nothing can
      // be marked on it: the size of the preview would put it elsewhere.
      let w = 0,
        h = 0;
      const m = sizeEl.match(/(\d+)[^\d]+(\d+)/);
      if (m) {
        w = parseInt(m[1], 10);
        h = parseInt(m[2], 10);
      }
      return { fileid, etag, w, h };
    },

    onRect(rect: Rect | null) {
      this.rect = rect;
      if (rect) {
        this.selectedFaceId = null;
        this.saveError = '';
        this.focusField('nameField');
      }
    },

    selectFace(faceId: number) {
      // While saving, the rectangle has to stay, to try again if it fails.
      if (this.saving) return;
      const face = this.faces.find((f) => f.id === faceId);
      if (!face) return;
      this.rect = null;
      this.saveError = '';
      this.selectedFaceId = faceId;
      this.editName = face.personName ?? '';
      // Only this face, unless the user says otherwise.
      this.wholeGroup = false;
      this.focusField('editField');
    },

    cancelEdit() {
      this.selectedFaceId = null;
      this.editName = '';
      this.saveError = '';
    },

    /** The rectangle as the server takes it, measured against the original. */
    manualRect() {
      const rect = this.rect!;
      return {
        fileId: this.fileId,
        x: rect.x,
        y: rect.y,
        width: rect.w,
        height: rect.h,
        imageWidth: this.imageNatW,
        imageHeight: this.imageNatH,
      };
    },

    saveNamed(): Promise<void> {
      return this.canSaveNamed ? this.saveMarking(this.name) : Promise.resolve();
    },

    saveUnnamed(): Promise<void> {
      return this.canSaveUnnamed ? this.saveMarking('') : Promise.resolve();
    },

    /**
     * Saves the rectangle as a face, and keeps the dialog open for the next
     * one. If saving fails, the rectangle stays, to try again.
     */
    async saveMarking(personName: string): Promise<void> {
      if (!this.rect || !this.fileId || !this.dimensionsKnown || this.saving) return;
      this.saving = true;
      this.saveError = '';
      try {
        await faceRecognitionAddManualFace({ ...this.manualRect(), personName });
        this.showNotice(
          personName
            ? t('memories', 'Person "{name}" tagged.', { name: personName })
            : t('memories', 'Saved. The face recognition will look for the person on its next run.'),
        );
        this.rect = null;
        this.rawInput = '';
        this.changed = true;
      } catch (e) {
        console.error(e);
        this.saveError = this.errorText(e, t('memories', 'Failed to save the manual face.'));
        return;
      } finally {
        this.saving = false;
      }
      await this.loadFaces();
    },

    async saveRegion(): Promise<void> {
      if (!this.rect || !this.fileId || !this.dimensionsKnown || this.saving || !this.canSearchRegion) return;
      this.saving = true;
      this.saveError = '';
      try {
        await faceRecognitionAddManualRegion(this.manualRect());
        this.showNotice(t('memories', 'The area will be searched for faces on the next run of the face recognition.'));
        this.rect = null;
        this.rawInput = '';
      } catch (e) {
        console.error(e);
        this.saveError = this.errorText(e, t('memories', 'The area could not be queued for a search.'));
        return;
      } finally {
        this.saving = false;
      }
      await this.loadFaces();
    },

    async saveEdit(): Promise<void> {
      const face = this.selectedFace;
      if (!this.canSaveEdit || !face || face.cluster === null) return;
      const target = this.editTarget;
      const wholeGroup = this.wholeGroup && this.canMoveGroup;
      this.saving = true;
      this.saveError = '';
      try {
        await faceRecognitionAssignCluster(face.cluster, target, wholeGroup ? undefined : face.id);
        this.showNotice(
          wholeGroup
            ? t('memories', 'Group assigned to "{name}".', { name: target })
            : t('memories', 'Face reassigned to "{name}".', { name: target }),
        );
        this.selectedFaceId = null;
        this.editName = '';
        this.changed = true;
      } catch (e) {
        console.error(e);
        this.saveError = this.errorText(e, t('memories', 'Failed to reassign the face.'));
        return;
      } finally {
        this.saving = false;
      }
      await this.loadFaces();
    },

    /** What went wrong with a call to the server, in words. */
    errorText(e: unknown, fallback: string): string {
      const response = responseOf(e);
      if (response?.status === 503) {
        return t('memories', 'Face Recognition on the server has to be updated first.');
      }
      if (response?.status === 409) {
        return t('memories', 'The face has moved to another group in the meantime. Please check it and try again.');
      }
      const detail = response?.data?.error;
      return detail ? `${fallback} (${String(detail)})` : fallback;
    },
  },
});
</script>

<style lang="scss" scoped>
.manual-face-add {
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-width: 320px;
}

.picker-step {
  padding: 16px 0;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 12px;
  align-items: center;
}

.draw-step {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.hint {
  font-size: 0.9em;
  opacity: 0.8;
  margin: 0;
}

.stage-wrap {
  position: relative;
}

// Over the photo, so that it moves nothing while it comes and goes.
.saved-notice {
  position: absolute;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  max-width: calc(100% - 24px);
  padding: 8px 14px;
  border-radius: var(--border-radius-large, 10px);
  background: var(--color-success, #2d7b41);
  color: var(--color-primary-element-text, #fff);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  font-size: 0.95em;
  text-align: center;
  pointer-events: none;
  z-index: 2;
}

.saved-notice-enter-active,
.saved-notice-leave-active {
  transition: opacity 0.25s ease;
}

.saved-notice-enter,
.saved-notice-leave-to {
  opacity: 0;
}

.legend {
  display: flex;
  flex-wrap: wrap;
  gap: 4px 14px;
  justify-content: center;
  font-size: 0.85em;
  opacity: 0.85;

  span {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }

  .swatch {
    display: inline-block;
    width: 16px;
    height: 10px;
    border: 2px solid #95a5a6;

    &.origin-auto {
      border-color: #2ecc71;
    }
    &.origin-manual {
      border-color: #f1c40f;
    }
    &.line-pending {
      border-style: dashed;
    }
    &.line-excluded {
      border-style: dotted;
      border-width: 3px;
    }
  }
}

.regions {
  font-size: 0.9em;

  .section-title {
    font-weight: bold;
    margin-bottom: 2px;
  }

  ul {
    margin: 0;
    padding-left: 18px;
    list-style: disc;
  }
}

.fields {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 6px;

  .face-title {
    font-weight: bold;
  }

  .state,
  .group {
    margin: 0;
    font-size: 0.9em;
  }

  .hint-text {
    color: var(--color-warning-text, var(--color-warning));
  }

  .scope {
    margin: 0;
    font-size: 0.9em;
    padding: 6px 8px;
    border-left: 3px solid var(--color-primary-element);
    background: var(--color-background-hover);
  }
}
</style>
