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
    class="c-lad-table-wrapper u-style-receiver js-style-receiver"
    :class="staleClass"
>
    <table class="c-table c-lad-table">
        <thead>
            <tr>
                <th>Name</th>
                <th>Timestamp</th>
                <th>Value</th>
                <th v-if="hasUnits">Unit</th>
            </tr>
        </thead>
        <tbody>
            <template
                v-for="ladTable in ladTableObjects"
            >
                <tr
                    :key="ladTable.key"
                    class="c-table__group-header js-lad-table-set__table-headers"
                >
                    <td colspan="10">
                        {{ ladTable.domainObject.name }}
                    </td>
                </tr>
                <lad-row
                    v-for="ladRow in ladTelemetryObjects[ladTable.key]"
                    :key="ladRow.key"
                    :domain-object="ladRow.domainObject"
                    :path-to-table="ladTable.objectPath"
                    :has-units="hasUnits"
                    :is-stale="staleObjects.includes(ladRow.key)"
                    @rowContextClick="updateViewContext"
                />
            </template>
        </tbody>
    </table>
</div>
</template>

<script>

import LadRow from './LADRow.vue';
import StalenessUtils from '@/utils/staleness';

export default {
    components: {
        LadRow
    },
    inject: ['openmct', 'objectPath', 'currentView'],
    props: {
        domainObject: {
            type: Object,
            required: true
        }
    },
    data() {
        return {
            ladTableObjects: [],
            ladTelemetryObjects: {},
            compositions: [],
            viewContext: {},
            staleObjects: []
        };
    },
    computed: {
        hasUnits() {
            let ladTables = Object.values(this.ladTelemetryObjects);
            for (let ladTable of ladTables) {
                for (let telemetryObject of ladTable) {
                    let metadata = this.openmct.telemetry.getMetadata(telemetryObject.domainObject);

                    if (metadata) {
                        for (let metadatum of metadata.valueMetadatas) {
                            if (metadatum.unit) {
                                return true;
                            }
                        }
                    }
                }
            }

            return false;
        },
        staleClass() {
            if (this.staleObjects.length !== 0) {
                return 'is-stale';
            }

            return '';
        }
    },
    mounted() {
        this.composition = this.openmct.composition.get(this.domainObject);
        this.composition.on('add', this.addLadTable);
        this.composition.on('remove', this.removeLadTable);
        this.composition.on('reorder', this.reorderLadTables);
        this.composition.load();

        this.stalenessSubscription = {};
    },
    destroyed() {
        this.composition.off('add', this.addLadTable);
        this.composition.off('remove', this.removeLadTable);
        this.composition.off('reorder', this.reorderLadTables);
        this.compositions.forEach(c => {
            c.composition.off('add', c.addCallback);
            c.composition.off('remove', c.removeCallback);
        });

        Object.values(this.stalenessSubscription).forEach(stalenessSubscription => {
            stalenessSubscription.unsubscribe();
            stalenessSubscription.stalenessUtils.destroy();
        });
    },
    methods: {
        addLadTable(domainObject) {
            let ladTable = {};
            ladTable.domainObject = domainObject;
            ladTable.key = this.openmct.objects.makeKeyString(domainObject.identifier);
            ladTable.objectPath = [domainObject, ...this.objectPath];

            this.$set(this.ladTelemetryObjects, ladTable.key, []);
            this.ladTableObjects.push(ladTable);

            let composition = this.openmct.composition.get(ladTable.domainObject);
            let addCallback = this.addTelemetryObject(ladTable);
            let removeCallback = this.removeTelemetryObject(ladTable);

            composition.on('add', addCallback);
            composition.on('remove', removeCallback);
            composition.load();

            this.compositions.push({
                composition,
                addCallback,
                removeCallback
            });
        },
        removeLadTable(identifier) {
            let index = this.ladTableObjects.findIndex(ladTable => this.openmct.objects.makeKeyString(identifier) === ladTable.key);
            let ladTable = this.ladTableObjects[index];

            this.$delete(this.ladTelemetryObjects, ladTable.key);
            this.ladTableObjects.splice(index, 1);
        },
        reorderLadTables(reorderPlan) {
            let oldComposition = this.ladTableObjects.slice();
            reorderPlan.forEach(reorderEvent => {
                this.$set(this.ladTableObjects, reorderEvent.newIndex, oldComposition[reorderEvent.oldIndex]);
            });
        },
        addTelemetryObject(ladTable) {
            return (domainObject) => {
                let telemetryObject = {};
                telemetryObject.key = this.openmct.objects.makeKeyString(domainObject.identifier);
                telemetryObject.domainObject = domainObject;

                let telemetryObjects = this.ladTelemetryObjects[ladTable.key];
                telemetryObjects.push(telemetryObject);

                this.$set(this.ladTelemetryObjects, ladTable.key, telemetryObjects);

                // if tracking already, possibly in another table, return
                if (this.stalenessSubscription[telemetryObject.key]) {
                    return;
                } else {
                    this.stalenessSubscription[telemetryObject.key] = {};
                    this.stalenessSubscription[telemetryObject.key].stalenessUtils = new StalenessUtils(this.openmct, domainObject);
                }

                this.openmct.telemetry.isStale(domainObject).then((stalenessResponse) => {
                    if (stalenessResponse !== undefined) {
                        this.handleStaleness(telemetryObject.key, stalenessResponse);
                    }
                });
                const stalenessSubscription = this.openmct.telemetry.subscribeToStaleness(domainObject, (stalenessResponse) => {
                    this.handleStaleness(telemetryObject.key, stalenessResponse);
                });

                this.stalenessSubscription[telemetryObject.key].unsubscribe = stalenessSubscription;
            };
        },
        removeTelemetryObject(ladTable) {
            return (identifier) => {
                const SKIP_CHECK = true;
                const keystring = this.openmct.objects.makeKeyString(identifier);
                let telemetryObjects = this.ladTelemetryObjects[ladTable.key];
                let index = telemetryObjects.findIndex(telemetryObject => keystring === telemetryObject.key);

                telemetryObjects.splice(index, 1);

                this.$set(this.ladTelemetryObjects, ladTable.key, telemetryObjects);

                this.stalenessSubscription[keystring].unsubscribe();
                this.stalenessSubscription[keystring].stalenessUtils.destroy();
                this.handleStaleness(keystring, { isStale: false }, SKIP_CHECK);
            };
        },
        handleStaleness(id, stalenessResponse, skipCheck = false) {
            if (skipCheck || this.stalenessSubscription[id].stalenessUtils.shouldUpdateStaleness(stalenessResponse)) {
                const index = this.staleObjects.indexOf(id);
                if (stalenessResponse.isStale) {
                    if (index === -1) {
                        this.staleObjects.push(id);
                    }
                } else {
                    if (index !== -1) {
                        this.staleObjects.splice(index, 1);
                    }
                }
            }
        },
        updateViewContext(rowContext) {
            this.viewContext.row = rowContext;
        },
        getViewContext() {
            return this.viewContext;
        }
    }
};
</script>
