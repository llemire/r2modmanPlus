<template>
    <div>
        <hero :title=heroTitle :subtitle='loadingText' :heroType=heroType />
        <div class='notification is-warning'>
            <p>Game updates may break mods. If a new update has been released, please be patient.</p>
        </div>
        <progress-bar
            :max='requests.length * 100'
            :value='reduceRequests().getProgress() > 0 ? reduceRequests().getProgress() : undefined'
            :className='[reduceRequests().getProgress() > 0 ? "is-info" : ""]' />
        <div class='columns'>
            <div class='column is-one-quarter'>
                <aside class='menu'>
                    <p class='menu-label'>Help</p>
                    <ul class='menu-list'>
                        <li><a @click="view = 'about'" :class="[view === 'about' ? 'is-active' : '']">About</a></li>
                        <li><a @click="view = 'faq'" :class="[view === 'faq' ? 'is-active' : '']">FAQ</a></li>
                        <li>
                            <link-component :url="'https://github.com/ebkr/r2modmanPlus'" :target="'external'">
                                <i class='fab fa-github fa-lg' aria-hidden='true' />
                            </link-component>
                        </li>
                    </ul>
                </aside>
            </div>
            <div class='column is-three-quarters'>
                <div class='container'>
                    <br />
                    <nav class='level' v-if='isOffline'>
                        <div class='level-item'>
                            <a class='button is-info margin-right margin-right--half-width' @click='continueOffline()'>Continue offline</a>
                            <a class='button' @click='retryConnection()'>Try to reconnect</a>
                        </div>
                        <br /><br />
                    </nav>
                </div>
                <div>
                    <article class='media'>
                        <div class='media-content'>
                            <div class='content'>
                                <div class='container' v-if="view !== 'main'">
                                    <i class='fas fa-long-arrow-alt-left margin-right' />
                                    <strong><a @click="view = 'main'">Go back</a></strong>
                                    <br /><br />
                                </div>
                                <div class='container' v-if="view === 'main'">
                                    <p>
                    <span class='icon margin-right margin-right--half-width'>
                      <i class='fas fa-info-circle' />
                    </span>
                                        <strong>Did you know?</strong>
                                    </p>
                                    <ul class='margin-right'>
                                        <li>
                                            <p>
                                                You can use the "Install with Mod Manager" button on
                                                <link-component
                                                    :url="'https://thunderstore.io'" :target="'external'">Thunderstore
                                                </link-component>
                                                with r2modman.
                                            </p>
                                        </li>
                                        <li>
                                            <p>
                                                You can export the selected profile from the settings screen as either a
                                                file, or a code.
                                                This makes it easy to share your mod list with friends!
                                            </p>
                                        </li>
                                    </ul>
                                    <p>
                    <span class='icon margin-right margin-right--half-width'>
                      <i class='fas fa-question-circle' />
                    </span>
                                        <strong>Having trouble?</strong>
                                    </p>
                                    <p>
                                        Send a screenshot of the error in the Thunderstore modding discord server. Feel
                                        free to ping me
                                        if it doesn't get resolved.
                                    </p>
                                </div>
                                <div class='container' v-else-if="view === 'about'">
                                    <p>
                    <span class='icon margin-right margin-right--half-width'>
                      <i class='fas fa-address-card' />
                    </span>
                                        <strong>About r2modman</strong>
                                    </p>
                                    <p>It's created by Ebkr, using Quasar.</p>
                                    <p>Quasar provides the following development tools that r2modman is built upon:</p>
                                    <ul>
                                        <li>Electron</li>
                                        <li>Node</li>
                                        <li>Vue</li>
                                        <li>TypeScript</li>
                                    </ul>
                                </div>
                                <div class='container' v-else-if="view === 'faq'">
                                    <p>
                    <span class='icon margin-right margin-right--half-width'>
                      <i class='fas fa-question-circle' />
                    </span>
                                        <strong>FAQ</strong>
                                    </p>
                                    <ul>
                                        <li>
                                            <p><strong>How do I get started?</strong></p>
                                            <p>
                                                Head on over to the online tab, and download BepInEx and R2API.
                                            </p>
                                        </li>
                                        <li>
                                            <p><strong>Starting the game with mods</strong></p>
                                            <p>
                                                You have to start the game from within the manager. Starting through
                                                Steam will not work
                                                without taking manual steps.
                                            </p>
                                        </li>
                                    </ul>
                                </div>
                            </div>
                        </div>
                    </article>
                </div>
            </div>
        </div>
    </div>
</template>

<script lang='ts'>
import Vue from 'vue';
import Component from 'vue-class-component';
import { Hero, Link, Progress } from '../components/all';

import RequestItem from '../model/requests/RequestItem';
import axios from 'axios';
import * as path from 'path';
import Profile from '../model/Profile';

import ThunderstorePackages from '../r2mm/data/ThunderstorePackages';
import { ipcRenderer } from 'electron';
import GameManager from '../model/game/GameManager';
import ApiCacheUtils from '../utils/ApiCacheUtils';
import GameDirectoryResolverProvider from '../providers/ror2/game/GameDirectoryResolverProvider';
import LinuxGameDirectoryResolver from '../r2mm/manager/linux/GameDirectoryResolver';
import FsProvider from '../providers/generic/file/FsProvider';
import PathResolver from '../r2mm/manager/PathResolver';
import ConnectionProvider from '../providers/generic/connection/ConnectionProvider';

@Component({
    components: {
        'hero': Hero,
        'progress-bar': Progress,
        'link-component': Link
    }
})
export default class Splash extends Vue {
    heroTitle: string = 'Starting r2modman';
    loadingText: string = 'Initialising';
    heroType: string = 'is-info';
    view: string = 'main';
    activeGame = GameManager.activeGame;

    requests = [
        new RequestItem('UpdateCheck', 0),
        new RequestItem('ThunderstoreDownload', 0),
        new RequestItem('ExclusionsList', 0)
    ];
    isOffline: boolean = false;

    exclusionMap: Map<string, boolean> = new Map();

    // Used to produce a single, combined, RequestItem./
    private reduceRequests(): RequestItem {
        return this.requests.reduce((x, y) => x.merge(y));
    }

    // Ensure that r2modman isn't outdated.
    private checkForUpdates() {
        this.loadingText = 'Preparing';
        ipcRenderer.once('update-done', () => {
            this.getRequestItem('UpdateCheck').setProgress(100);
            this.getExclusions();
        });
        ipcRenderer.send('update-app');
    }

    private async getExclusions() {
        this.loadingText = 'Connecting to GitHub repository';
        ConnectionProvider.instance.getExclusions(percentageProgress => {
            this.loadingText = 'Downloading exclusions';
            this.getRequestItem('ExclusionsList').setProgress(percentageProgress);
        }).then(exclusions => {
            this.exclusionMap = exclusions;
            ThunderstorePackages.EXCLUSIONS = this.exclusionMap;
            this.getThunderstoreMods(0);
        });
    }

    // Provide access to a request item, as item is not stored in a map.
    private getRequestItem(name: string): RequestItem {
        return this.requests.filter(ri => ri.getName() === name)[0];
    }

    // Get the list of Thunderstore mods via /api/v1/package.
    private async getThunderstoreMods(attempt: number) {
        this.loadingText = 'Connecting to Thunderstore';
        ApiCacheUtils.getLastRequest().then(async resp => {
            if (resp === undefined) {
                axios.get(this.activeGame.thunderstoreUrl, {
                    onDownloadProgress: progress => {
                        this.loadingText = 'Getting mod list from Thunderstore';
                        this.getRequestItem('ThunderstoreDownload').setProgress((progress.loaded / progress.total) * 100);
                    },
                    timeout: 30000
                }).then(async response => {
                    // Temporary. Creates a new standard profile until Profiles section is completed
                    new Profile('Default');
                    if (response.data === undefined) {
                        throw new Error("Response was undefined, retry for appropriate response.");
                    } else {
                        ThunderstorePackages.handlePackageApiResponse(response);
                        this.$store.dispatch('updateThunderstoreModList', ThunderstorePackages.PACKAGES);
                        await this.moveToNextScreen();
                    }
                }).catch((e_) => {
                    console.log(e_);
                    this.isOffline = true;
                    if (attempt < 3) {
                        this.getThunderstoreMods(attempt + 1);
                    } else {
                        this.heroTitle = 'Failed to get mods from Thunderstore';
                        this.loadingText = 'You may be offline, however you may still use R2MM offline.';
                    }
                });
            } else {
                ThunderstorePackages.handlePackageApiResponse({ data: resp.payload });
                if (ThunderstorePackages.EXCLUSIONS.size === 0) {
                    const exclusions = new Map<string, boolean>();
                    if (resp.exclusions) {
                        resp.exclusions.forEach(exclusion => {
                            exclusions.set(exclusion, true);
                        });
                    }
                    ThunderstorePackages.EXCLUSIONS = exclusions;
                }
                this.$store.dispatch("updateThunderstoreModList", ThunderstorePackages.PACKAGES);
                ThunderstorePackages.update(this.activeGame).then(() => {
                    this.$store.dispatch("updateThunderstoreModList", ThunderstorePackages.PACKAGES);
                });
                await this.moveToNextScreen();
            }
        });
    }

    async moveToNextScreen() {
        if (process.platform === 'linux') {
            if (!await (GameDirectoryResolverProvider.instance as LinuxGameDirectoryResolver).isProtonGame(this.activeGame)) {
                console.log('Not proton game');
                await this.ensureWrapperInGameFolder();
                const launchArgs = await (GameDirectoryResolverProvider.instance as LinuxGameDirectoryResolver).getLaunchArgs(this.activeGame);
                console.log(`Launch arguments for this game:`, launchArgs);
                if (typeof launchArgs === 'string' && !launchArgs.startsWith(path.join(PathResolver.MOD_ROOT, 'linux_wrapper.sh'))) {
                    this.$router.push({ path: '/linux-native-game-setup' });
                    return;
                }
            }
        } else if (process.platform === 'darwin') {
            await this.ensureWrapperInGameFolder();
            this.$router.push({ path: '/linux-native-game-setup' });
            return;
        }
        this.$router.push({ path: '/profiles' });
    }

    retryConnection() {
        this.$router.go(0);
    }

    continueOffline() {
        ThunderstorePackages.PACKAGES = [];
        this.moveToNextScreen();
    }

    private async ensureWrapperInGameFolder() {
        const wrapperName = process.platform === 'darwin' ? 'macos_proxy' : 'linux_wrapper.sh';
        console.log(`Ensuring wrapper for current game ${this.activeGame.displayName} in ${path.join(PathResolver.MOD_ROOT, wrapperName)}`);
        try {
            await FsProvider.instance.stat(path.join(PathResolver.MOD_ROOT, wrapperName));
            const oldBuf = (await FsProvider.instance.readFile(path.join(PathResolver.MOD_ROOT, wrapperName)));
            const newBuf = (await FsProvider.instance.readFile(path.join(__statics, wrapperName)));
            if (!oldBuf.equals(newBuf)) {
                throw new Error("Outdated buffer");
            }
        } catch (_) {
            if (await FsProvider.instance.exists(path.join(PathResolver.MOD_ROOT, wrapperName))) {
                await FsProvider.instance.unlink(path.join(PathResolver.MOD_ROOT, wrapperName));
            }
            await FsProvider.instance.copyFile(path.join(__statics, wrapperName), path.join(PathResolver.MOD_ROOT, wrapperName));
        }
        await FsProvider.instance.chmod(path.join(PathResolver.MOD_ROOT, wrapperName), 0o755);
    }

    async created() {
        this.loadingText = 'Checking for updates';
        setTimeout(this.checkForUpdates, 100);
    }
}
</script>
