<!-- eslint-disable vue/no-v-html -->
/*****************************************************************************
 * Open MCT, Copyright (c) 2014-2022, United States Government
 * as represented by the Administrator of the National Aeronautics and Space
 * Administration. All rights reserved.
 *
 * Open MCT is licensed under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 * http://www.apache.org/licenses/LICENSE-2.0.
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 *
 * Open MCT includes source code licensed under additional open source
 * licenses. See the Open Source Licenses file (LICENSES.md) included with
 * this source code distribution or the Licensing information page available
 * at runtime from the About dialog for additional information.
 *****************************************************************************/

<template>
<div
    class="c-notebook__entry c-ne has-local-controls"
    aria-label="Notebook Entry"
    :class="{ 'locked': isLocked, 'is-selected': isSelectedEntry }"
    @dragover="changeCursor"
    @drop.capture="cancelEditMode"
    @drop.prevent="dropOnEntry"
    @click="selectEntry($event, entry)"
>
    <div class="c-ne__time-and-content">
        <div class="c-ne__time-and-creator-and-delete">
            <div class="c-ne__time-and-creator">
                <span class="c-ne__created-date">{{ createdOnDate }}</span>
                <span class="c-ne__created-time">{{ createdOnTime }}</span>
                <span
                    v-if="entry.createdBy"
                    class="c-ne__creator"
                >
                    <span class="icon-person"></span> {{ entry.createdBy }}
                </span>
            </div>
            <span
                v-if="!readOnly && !isLocked"
                class="c-ne__local-controls--hidden"
            >
                <button
                    class="c-ne__remove c-icon-button c-icon-button--major icon-trash"
                    title="Delete this entry"
                    tabindex="-1"
                    @click="deleteEntry"
                >
                </button>
            </span>
        </div>
        <div class="c-ne__content">
            <template v-if="readOnly && result">
                <div
                    :id="entry.id"
                    class="c-ne__text highlight"
                    tabindex="0"
                >
                    <TextHighlight
                        :text="entryText"
                        :highlight="highlightText"
                        :highlight-class="'search-highlight'"
                    />
                </div>
            </template>
            <template v-else-if="!isLocked">
                <div
                    :id="entry.id"
                    class="c-ne__text c-ne__input"
                    aria-label="Notebook Entry Input"
                    tabindex="0"
                    :contenteditable="canEdit"
                    @mouseover="checkEditability($event)"
                    @mouseleave="canEdit = true"
                    @focus="editingEntry()"
                    @blur="updateEntryValue($event)"
                    @keydown.enter.exact.prevent
                    @keyup.enter.exact.prevent="forceBlur($event)"
                    v-html="formattedText"
                >
                </div>
            </template>

            <template v-else>
                <div
                    :id="entry.id"
                    class="c-ne__text"
                    contenteditable="false"
                    tabindex="0"
                    v-html="formattedText"
                >
                </div>
            </template>

            <div>
                <div
                    v-for="(tag, index) in entryTags"
                    :key="index"
                    class="c-tag"
                    :style="{ backgroundColor: tag.backgroundColor, color: tag.foregroundColor }"
                >
                    {{ tag.label }}
                </div>
            </div>

            <div
                :class="{'c-scrollcontainer': enableEmbedsWrapperScroll }"
            >
                <div
                    ref="embedsWrapper"
                    class="c-snapshots c-ne__embeds-wrapper"
                >
                    <NotebookEmbed
                        v-for="embed in entry.embeds"
                        ref="embeds"
                        :key="embed.id"
                        :embed="embed"
                        :is-locked="isLocked"
                        @removeEmbed="removeEmbed"
                        @updateEmbed="updateEmbed"
                    />
                </div>
            </div>
        </div>
    </div>
    <div
        v-if="readOnly"
        class="c-ne__section-and-page"
    >
        <a
            class="c-click-link"
            :class="{ 'search-highlight': result.metadata.sectionHit }"
            @click="navigateToSection()"
        >
            {{ result.section.name }}
        </a>
        <span class="icon-arrow-right"></span>
        <a
            class="c-click-link"
            :class="{ 'search-highlight': result.metadata.pageHit }"
            @click="navigateToPage()"
        >
            {{ result.page.name }}
        </a>
    </div>
</div>
</template>

<script>
import NotebookEmbed from './NotebookEmbed.vue';
import TextHighlight from '../../../utils/textHighlight/TextHighlight.vue';
import { createNewEmbed } from '../utils/notebook-entries';
import { saveNotebookImageDomainObject, updateNamespaceOfDomainObject } from '../utils/notebook-image';

import sanitizeHtml from 'sanitize-html';
import _ from 'lodash';

import Moment from 'moment';

const SANITIZATION_SCHEMA = {
    allowedTags: [],
    allowedAttributes: {}
};
const URL_REGEX = /https?:\/\/(www\.)?[-a-zA-Z0-9@:%._+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_+.~#?&//=]*)/g;
const UNKNOWN_USER = 'Unknown';

export default {
    components: {
        NotebookEmbed,
        TextHighlight
    },
    inject: ['openmct', 'snapshotContainer', 'entryUrlWhitelist'],
    props: {
        domainObject: {
            type: Object,
            default() {
                return {};
            }
        },
        notebookAnnotations: {
            type: Array,
            default() {
                return [];
            }
        },
        entry: {
            type: Object,
            default() {
                return {};
            }
        },
        result: {
            type: Object,
            default() {
                return {};
            }
        },
        selectedPage: {
            type: Object,
            default() {
                return {};
            }
        },
        selectedSection: {
            type: Object,
            default() {
                return {};
            }
        },
        readOnly: {
            type: Boolean,
            default() {
                return true;
            }
        },
        isLocked: {
            type: Boolean,
            default() {
                return false;
            }
        },
        selectedEntryId: {
            type: String,
            required: true
        }
    },
    data() {
        return {
            editMode: false,
            canEdit: true,
            enableEmbedsWrapperScroll: false
        };
    },
    computed: {
        createdOnDate() {
            return this.formatTime(this.entry.createdOn, 'YYYY-MM-DD');
        },
        createdOnTime() {
            return this.formatTime(this.entry.createdOn, 'HH:mm:ss');
        },
        formattedText() {
            // remove ANY tags
            let text = sanitizeHtml(this.entry.text, SANITIZATION_SCHEMA);

            if (this.editMode || !this.urlWhitelist) {
                return text;
            }

            text = text.replace(URL_REGEX, (match) => {
                const url = new URL(match);
                const domain = url.hostname;
                let result = match;
                let isMatch = this.urlWhitelist.find((partialDomain) => {
                    return domain.endsWith(partialDomain);
                });

                if (isMatch) {
                    result = `<a class="c-hyperlink" target="_blank" href="${match}">${match}</a>`;
                }

                return result;
            });

            return text;
        },
        isSelectedEntry() {
            return this.selectedEntryId === this.entry.id;
        },
        entryTags() {
            const tagsFromAnnotations = this.openmct.annotation.getTagsFromAnnotations(this.notebookAnnotations);

            return tagsFromAnnotations;
        },
        entryText() {
            let text = this.entry.text;

            if (!this.result.metadata.entryHit) {
                text = `[ no result for '${this.result.metadata.originalSearchText}' in entry ]`;
            }

            return text;
        },
        highlightText() {
            let text = '';

            if (this.result.metadata.entryHit) {
                text = this.result.metadata.originalSearchText;
            }

            return text;
        }
    },
    mounted() {
        this.manageEmbedLayout = _.debounce(this.manageEmbedLayout, 400);

        if (this.$refs.embedsWrapper) {
            this.embedsWrapperResizeObserver = new ResizeObserver(this.manageEmbedLayout);
            this.embedsWrapperResizeObserver.observe(this.$refs.embedsWrapper);
        }

        this.manageEmbedLayout();
        this.dropOnEntry = this.dropOnEntry.bind(this);
        if (this.entryUrlWhitelist?.length > 0) {
            this.urlWhitelist = this.entryUrlWhitelist;
        }
    },
    beforeDestroy() {
        if (this.embedsWrapperResizeObserver) {
            this.embedsWrapperResizeObserver.unobserve(this.$refs.embedsWrapper);
        }
    },
    methods: {
        async addNewEmbed(objectPath) {
            const bounds = this.openmct.time.bounds();
            const snapshotMeta = {
                bounds,
                link: null,
                objectPath,
                openmct: this.openmct
            };
            const newEmbed = await createNewEmbed(snapshotMeta);
            this.entry.embeds.push(newEmbed);

            this.manageEmbedLayout();
        },
        cancelEditMode(event) {
            const isEditing = this.openmct.editor.isEditing();
            if (isEditing) {
                this.openmct.editor.cancel();
            }
        },
        changeCursor(event) {
            event.preventDefault();

            if (!this.isLocked) {
                event.dataTransfer.dropEffect = 'copy';
            } else {
                event.dataTransfer.dropEffect = 'none';
                event.dataTransfer.effectAllowed = 'none';
            }
        },
        checkEditability($event) {
            if ($event.target.nodeName === 'A') {
                this.canEdit = false;
            }
        },
        deleteEntry() {
            this.$emit('deleteEntry', this.entry.id);
        },
        manageEmbedLayout() {
            if (this.$refs.embeds) {
                const embedsWrapperLength = this.$refs.embedsWrapper.clientWidth;
                const embedsTotalWidth = this.$refs.embeds.reduce((total, embed) => {
                    return embed.$el.clientWidth + total;
                }, 0);

                this.enableEmbedsWrapperScroll = embedsTotalWidth > embedsWrapperLength;
            }

        },
        async dropOnEntry($event) {
            $event.stopImmediatePropagation();

            const snapshotId = $event.dataTransfer.getData('openmct/snapshot/id');
            if (snapshotId.length) {
                const snapshot = this.snapshotContainer.getSnapshot(snapshotId);
                this.entry.embeds.push(snapshot.embedObject);
                this.snapshotContainer.removeSnapshot(snapshotId);

                const namespace = this.domainObject.identifier.namespace;
                const notebookImageDomainObject = updateNamespaceOfDomainObject(snapshot.notebookImageDomainObject, namespace);
                saveNotebookImageDomainObject(this.openmct, notebookImageDomainObject);
            } else {
                const data = $event.dataTransfer.getData('openmct/domain-object-path');
                const objectPath = JSON.parse(data);
                await this.addNewEmbed(objectPath);
            }

            this.timestampAndUpdate();
        },
        findPositionInArray(array, id) {
            let position = -1;
            array.some((item, index) => {
                const found = item.id === id;
                if (found) {
                    position = index;
                }

                return found;
            });

            return position;
        },
        forceBlur(event) {
            event.target.blur();
        },
        formatTime(unixTime, timeFormat) {
            return Moment.utc(unixTime).format(timeFormat);
        },
        navigateToPage() {
            this.$emit('changeSectionPage', {
                sectionId: this.result.section.id,
                pageId: this.result.page.id
            });
        },
        navigateToSection() {
            this.$emit('changeSectionPage', {
                sectionId: this.result.section.id,
                pageId: null
            });
        },
        removeEmbed(id) {
            const embedPosition = this.findPositionInArray(this.entry.embeds, id);
            // TODO: remove notebook snapshot object using object remove API
            this.entry.embeds.splice(embedPosition, 1);

            this.timestampAndUpdate();

            this.manageEmbedLayout();
        },
        updateEmbed(newEmbed) {
            this.entry.embeds.some(e => {
                const found = (e.id === newEmbed.id);
                if (found) {
                    e = newEmbed;
                }

                return found;
            });

            this.timestampAndUpdate();
        },
        async timestampAndUpdate() {
            const user = await this.openmct.user.getCurrentUser();

            if (user === undefined) {
                this.entry.modifiedBy = UNKNOWN_USER;
            }

            this.entry.modified = Date.now();

            this.$emit('updateEntry', this.entry);
        },
        editingEntry() {
            this.editMode = true;
            this.$emit('editingEntry');
        },
        updateEntryValue($event) {
            this.editMode = false;
            const value = $event.target.innerText;
            if (value !== this.entry.text && value.match(/\S/)) {
                this.entry.text = value;
                this.timestampAndUpdate();
            } else {
                this.$emit('cancelEdit');
            }
        },
        selectEntry(event, entry) {
            const targetDetails = {};
            const keyString = this.openmct.objects.makeKeyString(this.domainObject.identifier);
            targetDetails[keyString] = {
                entryId: entry.id
            };
            const targetDomainObjects = {};
            targetDomainObjects[keyString] = this.domainObject;
            this.openmct.selection.select(
                [
                    {
                        element: this.openmct.layout.$refs.browseObject.$el,
                        context: {
                            item: this.domainObject
                        }
                    },
                    {
                        element: event.currentTarget,
                        context: {
                            type: 'notebook-entry-selection',
                            targetDetails,
                            targetDomainObjects,
                            annotations: this.notebookAnnotations,
                            annotationType: this.openmct.annotation.ANNOTATION_TYPES.NOTEBOOK,
                            onAnnotationChange: this.timestampAndUpdate
                        }
                    }
                ],
                false);
            event.stopPropagation();
            this.$emit('entry-selection', this.entry);
        }
    }
};
</script>
