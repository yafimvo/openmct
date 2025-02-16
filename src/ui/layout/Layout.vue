<template>
<div
    class="l-shell"
    :class="{
        'is-editing': isEditing
    }"
>

    <div
        id="splash-screen"
    ></div>

    <div
        class="l-shell__head"
        :class="{
            'l-shell__head--expanded': headExpanded,
            'l-shell__head--minify-indicators': !headExpanded
        }"
    >
        <CreateButton class="l-shell__create-button" />
        <GrandSearch
            ref="grand-search"
        />
        <indicators class="l-shell__head-section l-shell__indicators" />
        <button
            class="l-shell__head__collapse-button c-icon-button"
            :class="headExpanded ? 'l-shell__head__collapse-button--collapse' : 'l-shell__head__collapse-button--expand'"
            :title="`Click to ${headExpanded ? 'collapse' : 'expand'} items`"
            @click="toggleShellHead"
        ></button>
        <notification-banner />
        <div class="l-shell__head-section l-shell__controls">
            <button
                class="c-icon-button c-icon-button--major icon-new-window"
                title="Open in a new browser tab"
                target="_blank"
                @click="openInNewTab"
            ></button>
            <button
                :class="['c-icon-button c-icon-button--major', fullScreen ? 'icon-fullscreen-collapse' : 'icon-fullscreen-expand']"
                :title="`${fullScreen ? 'Exit' : 'Enable'} full screen mode`"
                @click="fullScreenToggle"
            ></button>
        </div>
        <app-logo />
    </div>

    <div class="l-shell__drawer c-drawer c-drawer--push c-drawer--align-top"></div>

    <multipane
        class="l-shell__main"
        :class="[resizingClass]"
        type="horizontal"
    >
        <pane
            class="l-shell__pane-tree"
            style="width: 300px;"
            handle="after"
            label="Browse"
            hide-param="hideTree"
            :persist-position="true"
            @start-resizing="onStartResizing"
            @end-resizing="onEndResizing"
        >
            <button
                slot="controls"
                class="c-icon-button l-shell__reset-tree-button icon-folders-collapse"
                title="Collapse all tree items"
                @click="handleTreeReset"
            >
            </button>
            <button
                slot="controls"
                class="c-icon-button l-shell__sync-tree-button icon-target"
                title="Show selected item in tree"
                @click="handleSyncTreeNavigation"
            >
            </button>
            <multipane
                type="vertical"
            >
                <pane
                    id="tree-pane"
                >
                    <mct-tree
                        ref="mctTree"
                        :sync-tree-navigation="triggerSync"
                        :reset-tree-navigation="triggerReset"
                        class="l-shell__tree"
                    />
                </pane>
                <pane
                    handle="before"
                    label="Recently Viewed"
                    :persist-position="true"
                >
                    <RecentObjectsList
                        class="l-shell__tree"
                        @openAndScrollTo="openAndScrollTo($event)"
                    />
                </pane>
            </multipane>
        </pane>
        <pane class="l-shell__pane-main">
            <browse-bar
                ref="browseBar"
                class="l-shell__main-view-browse-bar"
                :action-collection="actionCollection"
                @sync-tree-navigation="handleSyncTreeNavigation"
            />
            <toolbar
                v-if="toolbar"
                class="l-shell__toolbar"
            />
            <object-view
                ref="browseObject"
                class="l-shell__main-container js-main-container js-notebook-snapshot-item"
                data-selectable
                :show-edit-view="true"
                @change-action-collection="setActionCollection"
            />
            <component
                :is="conductorComponent"
                class="l-shell__time-conductor"
            />
        </pane>
        <pane
            class="l-shell__pane-inspector l-pane--holds-multipane"
            handle="before"
            label="Inspect"
            hide-param="hideInspector"
            :persist-position="true"
            @start-resizing="onStartResizing"
            @end-resizing="onEndResizing"
        >
            <Inspector
                ref="inspector"
                :is-editing="isEditing"
            />
        </pane>
    </multipane>
</div>
</template>

<script>
import Inspector from '../inspector/Inspector.vue';
import MctTree from './mct-tree.vue';
import ObjectView from '../components/ObjectView.vue';
import CreateButton from './CreateButton.vue';
import GrandSearch from './search/GrandSearch.vue';
import multipane from './multipane.vue';
import pane from './pane.vue';
import BrowseBar from './BrowseBar.vue';
import Toolbar from '../toolbar/Toolbar.vue';
import AppLogo from './AppLogo.vue';
import Indicators from './status-bar/Indicators.vue';
import NotificationBanner from './status-bar/NotificationBanner.vue';
import RecentObjectsList from './RecentObjectsList.vue';

export default {
    components: {
        Inspector,
        MctTree,
        ObjectView,
        CreateButton,
        GrandSearch,
        multipane,
        pane,
        BrowseBar,
        Toolbar,
        AppLogo,
        Indicators,
        NotificationBanner,
        RecentObjectsList
    },
    inject: ['openmct'],
    data: function () {
        let storedHeadProps = window.localStorage.getItem('openmct-shell-head');
        let headExpanded = true;
        if (storedHeadProps) {
            headExpanded = JSON.parse(storedHeadProps).expanded;
        }

        return {
            fullScreen: false,
            conductorComponent: undefined,
            isEditing: false,
            hasToolbar: false,
            actionCollection: undefined,
            triggerSync: false,
            triggerReset: false,
            headExpanded,
            isResizing: false
        };
    },
    computed: {
        toolbar() {
            return this.hasToolbar && this.isEditing;
        },
        resizingClass() {
            return this.isResizing ? 'l-shell__resizing' : '';
        }
    },
    mounted() {
        this.openmct.editor.on('isEditing', (isEditing) => {
            this.isEditing = isEditing;
        });

        this.openmct.selection.on('change', this.toggleHasToolbar);
    },
    methods: {
        enterFullScreen() {
            let docElm = document.documentElement;

            if (docElm.requestFullscreen) {
                docElm.requestFullscreen();
            } else if (docElm.mozRequestFullScreen) { /* Firefox */
                docElm.mozRequestFullScreen();
            } else if (docElm.webkitRequestFullscreen) { /* Chrome, Safari and Opera */
                docElm.webkitRequestFullscreen();
            } else if (docElm.msRequestFullscreen) { /* IE/Edge */
                docElm.msRequestFullscreen();
            }
        },
        exitFullScreen() {
            if (document.exitFullscreen) {
                document.exitFullscreen();
            } else if (document.mozCancelFullScreen) {
                document.mozCancelFullScreen();
            } else if (document.webkitCancelFullScreen) {
                document.webkitCancelFullScreen();
            } else if (document.msExitFullscreen) {
                document.msExitFullscreen();
            }
        },
        toggleShellHead() {
            this.headExpanded = !this.headExpanded;

            window.localStorage.setItem(
                'openmct-shell-head',
                JSON.stringify(
                    {
                        expanded: this.headExpanded
                    }
                )
            );
        },
        fullScreenToggle() {
            if (this.fullScreen) {
                this.fullScreen = false;
                this.exitFullScreen();
            } else {
                this.fullScreen = true;
                this.enterFullScreen();
            }
        },
        openInNewTab(event) {
            window.open(window.location.href);
        },
        toggleHasToolbar(selection) {
            let structure = undefined;

            if (!selection || !selection[0]) {
                structure = [];
            } else {
                structure = this.openmct.toolbars.get(selection);
            }

            this.hasToolbar = structure.length > 0;
        },
        openAndScrollTo(navigationPath) {
            this.$refs.mctTree.openAndScrollTo(navigationPath);
            this.$refs.mctTree.targetedPath = navigationPath;
        },
        setActionCollection(actionCollection) {
            this.actionCollection = actionCollection;
        },
        handleSyncTreeNavigation() {
            this.triggerSync = !this.triggerSync;
        },
        handleTreeReset() {
            this.triggerReset = !this.triggerReset;
        },
        onStartResizing() {
            this.isResizing = true;
        },
        onEndResizing() {
            this.isResizing = false;
        }
    }
};
</script>
