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
    class="c-gsearch-result c-gsearch-result--annotation"
    aria-label="Search Result"
    role="presentation"
>
    <div
        class="c-gsearch-result__type-icon"
        :class="resultTypeIcon"
    ></div>
    <div
        class="c-gsearch-result__body"
        aria-label="Annotation Search Result"
    >
        <div
            class="c-gsearch-result__title"
            @click="clickedResult"
        >
            {{ getResultName }}
        </div>

        <ObjectPath
            :domain-object="domainObject"
            :read-only="false"
            :show-object-itself="true"
        />

        <div class="c-gsearch-result__tags">
            <div
                v-for="(tag, index) in result.fullTagModels"
                :key="index"
                class="c-tag"
                :class="{ '--is-not-search-match': !isSearchMatched(tag) }"
                :style="{ backgroundColor: tag.backgroundColor, color: tag.foregroundColor }"
            >
                {{ tag.label }}
            </div>
        </div>
    </div>
    <div class="c-gsearch-result__more-options-button">
        <button class="c-icon-button icon-3-dots"></button>
    </div>
</div>
</template>

<script>
import ObjectPath from '../../components/ObjectPath.vue';
import { identifierToString } from '../../../../src/tools/url';

export default {
    name: 'AnnotationSearchResult',
    components: {
        ObjectPath
    },
    inject: ['openmct'],
    props: {
        result: {
            type: Object,
            required: true,
            default() {
                return {};
            }
        }
    },
    data() {
        return {
        };
    },
    computed: {
        domainObject() {
            return this.result.targetModels[0];
        },
        getResultName() {
            if (this.result.annotationType === this.openmct.annotation.ANNOTATION_TYPES.NOTEBOOK) {
                const targetID = Object.keys(this.result.targets)[0];
                const entryIdToFind = this.result.targets[targetID].entryId;
                const notebookModel = this.result.targetModels[0].configuration.entries;

                const sections = Object.values(notebookModel);
                for (const section of sections) {
                    const pages = Object.values(section);
                    for (const entries of pages) {
                        for (const entry of entries) {
                            if (entry.id === entryIdToFind) {
                                return entry.text;
                            }
                        }
                    }
                }

                return "Could not find any matching Notebook entries";
            } else {
                return this.result.targetModels[0].name;
            }
        },
        resultTypeIcon() {
            return this.openmct.types.get(this.result.type).definition.cssClass;
        },
        tagBackgroundColor() {
            return this.result.fullTagModels[0].backgroundColor;
        },
        tagForegroundColor() {
            return this.result.fullTagModels[0].foregroundColor;
        }
    },
    methods: {
        clickedResult() {
            const objectPath = this.domainObject.originalPath;
            let resultUrl = identifierToString(this.openmct, objectPath);

            this.openmct.router.navigate(resultUrl);
            if (this.result.annotationType === this.openmct.annotation.ANNOTATION_TYPES.PLOT_SPATIAL) {
                //wait a beat for the navigation
                setTimeout(() => {
                    this.clickedPlotAnnotation();
                }, 100);
            }
        },
        clickedPlotAnnotation() {
            const targetDetails = {};
            const targetDomainObjects = {};
            Object.entries(this.result.targets).forEach(([key, value]) => {
                targetDetails[key] = value;
            });
            this.result.targetModels.forEach((targetModel) => {
                const keyString = this.openmct.objects.makeKeyString(targetModel.identifier);
                targetDomainObjects[keyString] = targetModel;
            });
            const selection =
                    [
                        {
                            element: this.openmct.layout.$refs.browseObject.$el,
                            context: {
                                item: this.result
                            }
                        },
                        {
                            element: this.$el,
                            context: {
                                type: 'plot-points-selection',
                                targetDetails,
                                targetDomainObjects,
                                annotations: [this.result],
                                annotationType: this.openmct.annotation.ANNOTATION_TYPES.PLOT_SPATIAL,
                                onAnnotationChange: () => {}
                            }
                        }
                    ];
            this.openmct.selection.select(selection, true);
        },
        isSearchMatched(tag) {
            if (this.result.matchingTagKeys) {
                return this.result.matchingTagKeys.includes(tag.tagID);
            }

            return false;
        }
    }
};
</script>
