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
    class="c-inspector__properties c-inspect-properties has-tag-applier"
    aria-label="Tags Inspector"
>
    <div
        class="c-inspect-properties__header"
    >
        Tags
    </div>
    <div
        v-if="shouldShowTagsEditor"
        class="c-inspect-properties__section"
    >
        <TagEditor
            :targets="targetDetails"
            :target-domain-objects="targetDomainObjects"
            :domain-object="domainObject"
            :annotations="loadedAnnotations"
            :annotation-type="annotationType"
            :on-tag-change="onAnnotationChange"
        />
    </div>
    <div
        v-else
        class="c-inspect-properties__row--span-all"
    >
        {{ noTagsMessage }}
    </div>
</div>
</template>

<script>
import TagEditor from '../../components/tags/TagEditor.vue';
import _ from 'lodash';

export default {
    components: {
        TagEditor
    },
    inject: ['openmct'],
    data() {
        return {
            selection: null,
            lastLocalAnnotationCreations: {},
            unobserveEntries: {},
            loadedAnnotations: []
        };
    },
    computed: {
        hasAnnotations() {
            return Boolean(
                this.loadedAnnotations
                && this.loadedAnnotations.length
            );
        },
        nonTagAnnotations() {
            if (!this.loadedAnnotations) {
                return [];
            }

            return this.loadedAnnotations.filter(annotation => {
                return !annotation.tags && !annotation._deleted;
            });
        },
        tagAnnotations() {
            if (!this.loadedAnnotations) {
                return [];
            }

            return this.loadedAnnotations.filter(annotation => {
                return !annotation.tags && !annotation._deleted;
            });
        },
        multiSelection() {
            return this.selection && this.selection.length > 1;
        },
        noAnnotationsMessage() {
            return this.multiSelection
                ? 'No annotations to display for multiple items'
                : 'No annotations to display for this item';
        },
        noTagsMessage() {
            return this.multiSelection
                ? 'No tags to display for multiple items'
                : 'No tags to display for this item';
        },
        domainObject() {
            return this?.selection?.[0]?.[0]?.context?.item;
        },
        targetDetails() {
            return this?.selection?.[0]?.[1]?.context?.targetDetails ?? {};
        },
        shouldShowTagsEditor() {
            return Object.keys(this.targetDetails).length > 0;
        },
        targetDomainObjects() {
            return this?.selection?.[0]?.[1]?.context?.targetDomainObjects ?? {};
        },
        selectedAnnotations() {
            return this?.selection?.[0]?.[1]?.context?.annotations;
        },
        annotationType() {
            return this?.selection?.[0]?.[1]?.context?.annotationType;
        },
        annotationFilter() {
            return this?.selection?.[0]?.[1]?.context?.annotationFilter;
        },
        onAnnotationChange() {
            return this?.selection?.[0]?.[1]?.context?.onAnnotationChange;
        }
    },
    async mounted() {
        this.openmct.annotation.on('targetDomainObjectAnnotated', this.loadAnnotationForTargetObject);
        this.openmct.selection.on('change', this.updateSelection);
        await this.updateSelection(this.openmct.selection.get());
    },
    beforeDestroy() {
        this.openmct.selection.off('change', this.updateSelection);
        const unobserveEntryFunctions = Object.values(this.unobserveEntries);
        unobserveEntryFunctions.forEach(unobserveEntry => {
            unobserveEntry();
        });
    },
    methods: {
        loadNewAnnotations(annotationsToLoad) {
            if (!annotationsToLoad || !annotationsToLoad.length) {
                this.loadedAnnotations.splice(0);

                return;
            }

            const sortedAnnotations = annotationsToLoad.sort((annotationA, annotationB) => {
                return annotationB.modified - annotationA.modified;
            });

            const mutableAnnotations = sortedAnnotations.map((annotation) => {
                return this.openmct.objects.toMutable(annotation);
            });

            if (sortedAnnotations.length < this.loadedAnnotations.length) {
                this.loadedAnnotations = this.loadedAnnotations.slice(0, mutableAnnotations.length);
            }

            for (let index = 0; index < mutableAnnotations.length; index += 1) {
                this.$set(this.loadedAnnotations, index, mutableAnnotations[index]);
            }
        },
        updateSelection(selection) {
            const unobserveEntryFunctions = Object.values(this.unobserveEntries);
            unobserveEntryFunctions.forEach(unobserveEntry => {
                unobserveEntry();
            });
            this.unobserveEntries = {};

            this.selection = selection;
            const targetKeys = Object.keys(this.targetDomainObjects);
            targetKeys.forEach(targetKey => {
                const targetObject = this.targetDomainObjects[targetKey];
                this.lastLocalAnnotationCreations[targetKey] = targetObject?.annotationLastCreated ?? 0;
                if (!this.unobserveEntries[targetKey]) {
                    this.unobserveEntries[targetKey] = this.openmct.objects.observe(targetObject, '*', this.targetObjectChanged);
                }
            });
            this.loadNewAnnotations(this.selectedAnnotations);
        },
        async targetObjectChanged(target) {
            const targetID = this.openmct.objects.makeKeyString(target.identifier);
            const lastLocalAnnotationCreation = this.lastLocalAnnotationCreations[targetID] ?? 0;
            if (lastLocalAnnotationCreation < target.annotationLastCreated) {
                this.lastLocalAnnotationCreations[targetID] = target.annotationLastCreated;
                await this.loadAnnotationForTargetObject(target);
            }
        },
        async loadAnnotationForTargetObject(target) {
            const targetID = this.openmct.objects.makeKeyString(target.identifier);
            const allAnnotationsForTarget = await this.openmct.annotation.getAnnotations(target.identifier);
            const filteredAnnotationsForSelection = allAnnotationsForTarget.filter(annotation => {
                const matchingTargetID = Object.keys(annotation.targets).filter(loadedTargetID => {
                    return targetID === loadedTargetID;
                });
                const fetchedTargetDetails = annotation.targets[matchingTargetID];
                const selectedTargetDetails = this.targetDetails[matchingTargetID];

                return _.isEqual(fetchedTargetDetails, selectedTargetDetails);
            });
            this.loadNewAnnotations(filteredAnnotationsForSelection);
        }
    }
};
</script>
