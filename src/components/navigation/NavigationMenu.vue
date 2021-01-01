<template>
    <div class="sticky-top sticky-top--no-shadow sticky-top--no-padding">
        <div id='gameRunningModal' :class="['modal', {'is-active':(gameRunning !== false)}]">
            <div class="modal-background" @click="closeGameRunningModal()"></div>
            <div class='modal-content'>
                <div class='notification is-info'>
                    <h3 class='title'>Risk of Rain 2 is launching via Steam</h3>
                    <h5 class="title is-5">Close this message to continue modding.</h5>
                    <p>If this is taking a while, it's likely due to Steam starting.</p>
                    <p>Please be patient, and have fun!</p>
                </div>
            </div>
            <button class="modal-close is-large" aria-label="close" @click="closeGameRunningModal()"></button>
        </div>
        <aside class="menu">
            <p class="menu-label">Risk of Rain 2</p>
            <ul class="menu-list">
                <li><a href="#" @click="launchModded"><i class="fas fa-play-circle icon--margin-right"/>Start modded</a>
                </li>
                <li>
                    <a href="#" @click="launchVanilla"><i class="far fa-play-circle icon--margin-right"/>Start
                        vanilla</a>
                </li>
            </ul>
            <p class="menu-label">Mods</p>
            <ul class="menu-list">
                <li>
                    <!-- Strange bug seemingly caused by CSS Grid display. Children @click is not being passed up to parent. -->
                    <!-- Due to this, the click event must be applied to all children. Parent container also binds click to account for margins. -->
                    <a href="#" data-ref="installed" @click="emitClick($event.target)"
                       class="tagged-link" :class="[view === 'installed' ? 'is-active' : '']">
                        <i class="fas fa-folder tagged-link__icon icon--margin-right" data-ref="installed" @click="emitClick($event.target)"/>
                        <span class="tagged-link__content" data-ref="installed" @click="emitClick($event.target)">Installed</span>
                        <span class="tag tagged-link__tag" :class="[{'is-link': view !== 'installed'}]"
                        data-ref="installed" @click="emitClick($event.target)">{{localModList.length}}</span>
                    </a>
                </li>
                <li>
                    <a href="#" data-ref="online" @click="emitClick($event.target)"
                       class="tagged-link" :class="[view === 'online' ? 'is-active' : '']">
                        <i class="fas fa-globe tagged-link__icon icon--margin-right" data-ref="online" @click="emitClick($event.target)"/>
                        <span class="tagged-link__content" data-ref="online" @click="emitClick($event.target)">Online</span>
                        <span class="tag tagged-link__tag" :class="[{'is-link': view !== 'online'}]"
                              data-ref="online" @click="emitClick($event.target)">{{thunderstoreModList.length}}</span>
                    </a>
                </li>
            </ul>
            <p class='menu-label'>Other</p>
            <ul class='menu-list'>
                <li>
                    <a href="#" :class="[view === 'config-editor' ? 'is-active' : '']" data-ref="config-editor" @click="emitClick($event.target)">
                        <i class="fas fa-edit icon--margin-right"/>Config editor
                    </a>
                </li>
                <li>
                    <a href="#" :class="[view === 'settings' ? 'is-active' : '']"
                       data-ref="settings" @click="emitClick($event.target)">
                        <i class="fas fa-cog icon--margin-right"/>Settings
                    </a>
                </li>
                <li>
                    <a href="#" :class="[view === 'help' ? 'is-active' : '']"
                       data-ref="help" @click="emitClick($event.target)">
                        <i class="fas fa-question-circle icon--margin-right"/>Help</a>
                    <ul v-if="view === 'help'">
                        <li>
                            <a href='#' :class="[{'is-active': helpPage === 'tips-and-tricks'}]"
                               data-ref="tips-and-tricks" @click="emitHelpSectionClick($event.target)">
                                <i class="fas fa-lightbulb icon--margin-right"/>Tips and tricks
                            </a>
                        </li>
                        <li>
                            <a href='#' :class="[{'is-active': helpPage === 'game-wont-start'}]"
                               data-ref="game-wont-start" @click="emitHelpSectionClick($event.target)">
                                <i class="fas fa-gamepad icon--margin-right"/>Game won't start
                            </a>
                        </li>
                        <li>
                            <a href='#' :class="[{'is-active': helpPage === 'mods-not-working'}]"
                               data-ref="mods-not-working" @click="emitHelpSectionClick($event.target)">
                                <i class="fas fa-ban icon--margin-right"/>Mods aren't working
                            </a>
                        </li>
                        <li>
                            <a href='#' :class="[{'is-active': helpPage === 'like-r2'}]"
                               data-ref="like-r2" @click="emitHelpSectionClick($event.target)">
                                <i class="fas fa-heart icon--margin-right"/>Like {{ appName }}?
                            </a>
                        </li>
                    </ul>
                </li>
            </ul>
        </aside>
    </div>
</template>

<script lang="ts">

    import { Component, Prop, Vue } from 'vue-property-decorator';
    import R2Error from '../../model/errors/R2Error';
    import GameDirectoryResolver from '../../r2mm/manager/GameDirectoryResolver';
    import FsProvider from '../../providers/generic/file/FsProvider';
    import ModLinker from '../../r2mm/manager/ModLinker';
    import ManagerSettings from '../../r2mm/manager/ManagerSettings';
    import ManifestV2 from "../../model/ManifestV2";
    import GameRunnerProviderImpl from '../../providers/generic/game/GameRunnerProviderImpl';
    import ManagerInformation from '../../_managerinf/ManagerInformation';

    @Component
    export default class NavigationMenu extends Vue {

        @Prop({default: ""})
        private view!: string;

        @Prop({default: ""})
        private helpPage!: string;

        private gameRunning: boolean = false;

        get settings() {
            return ManagerSettings.getSingleton();
        }

        get thunderstoreModList() {
            return this.$store.state.thunderstoreModList || [];
        }

        get localModList(): ManifestV2[] {
            return this.$store.state.localModList || [];
        }

        get appName(): string {
            return ManagerInformation.APP_NAME;
        }

        emitClick(element: any) {
            this.$emit("clicked-" + element.getAttribute("data-ref"));
        }

        emitHelpSectionClick(element: HTMLElement) {
            this.$emit("help-clicked-" + element.getAttribute("data-ref"));
        }

        async prepareLaunch() {
            const settings = await this.settings;
            let dir: string | R2Error;
            if (settings.riskOfRain2Directory === null) {
                dir = await GameDirectoryResolver.getDirectory();
            } else {
                dir = settings.riskOfRain2Directory;
            }
            if (dir instanceof R2Error) {
                // Show folder selection dialog.
                this.$emit("error", dir);
            } else {
                const setInstallDirError: R2Error | void = await settings.setRiskOfRain2Directory(dir);
                if (setInstallDirError instanceof R2Error) {
                    this.$emit("error", setInstallDirError);
                    return;
                }
            }
        }

        async launchModded() {
            const fs = FsProvider.instance;
            const settings = await this.settings;
            await this.prepareLaunch();
            if (settings.riskOfRain2Directory !== null && await fs.exists(settings.riskOfRain2Directory)) {
                const newLinkedFiles = await ModLinker.link();
                if (newLinkedFiles instanceof R2Error) {
                    this.$emit("error", newLinkedFiles);
                    return;
                } else {
                    const saveError = await settings.setLinkedFiles(newLinkedFiles);
                    if (saveError instanceof R2Error) {
                        this.$emit("error", saveError);
                        return;
                    }
                }
                this.gameRunning = true;
                GameRunnerProviderImpl.instance.startModded().then(value => {
                    if (value instanceof R2Error) {
                        this.$emit("error", value);
                    }
                    this.gameRunning = false;
                });
            } else {
                const err = new R2Error('Failed to start Risk of Rain 2', 'The Risk of Rain 2 directory does not exist',
                    'Set the Risk of Rain 2 directory in the Settings screen');
                this.$emit("error", err);
            }
        }

        async launchVanilla() {
            const fs = FsProvider.instance;
            const settings = await this.settings;
            await this.prepareLaunch();
            if (settings.riskOfRain2Directory !== null && await fs.exists(settings.riskOfRain2Directory)) {
                this.gameRunning = true;
                GameRunnerProviderImpl.instance.startVanilla().then(value => {
                    if (value instanceof R2Error) {
                        this.$emit("error", value);
                    }
                    this.gameRunning = false;
                });
            } else {
                const err = new R2Error('Failed to start Risk of Rain 2', 'The Risk of Rain 2 directory does not exist',
                    'Set the Risk of Rain 2 directory in the Settings screen');
                this.$emit("error", err);
            }
        }

        closeGameRunningModal() {
            this.gameRunning = false;
        }
    }

</script>