<template>
    <v-card
        flat
        class="mv-cell d-flex"
        :class="{
            'edit-mode': pausedMode,
        }"
    >
        <!-- When Cell has no content: show video picker -->
        <v-sheet style="height: 100%" class="d-flex flex-column pt-4" v-if="!cellContent">
            <!--================= No Content Mode ================-->
            <v-list
                class="mx-6 thin-scroll-bar flex-grow-1 flex-shrink-1"
                v-if="!cellContent"
                style="overflow-y: auto; overflow-x: hidden; position: relative"
            >
                <v-sheet class="px-0 d-flex flex-grow-1 align-stretch mb-1">
                    <v-btn
                        class="flex-grow-1 mr-2"
                        color="indigo darken-1"
                        style="max-width: 300px; flex-basis: 50%"
                        @click="$emit('showSelector', item.i)"
                    >
                        <v-icon>{{ icons.mdiMagnify }}</v-icon>
                    </v-btn>
                    <v-btn
                        class="flex-grow-1"
                        color="teal darken-1"
                        style="max-width: 300px; flex-basis: 20%"
                        @click="setItemAsChat(item)"
                        v-if="!(cellContent && cellContent.type === 'chat')"
                    >
                        <v-icon>{{ mdiMessage }}</v-icon>
                    </v-btn>
                </v-sheet>
            </v-list>
            <template>
                <CellControl :playIcon="icons.mdiPlay" @delete="deleteCell" class="mx-6 mb-6 mt-0 flex-grow-0" />
            </template>
        </v-sheet>

        <!--=== Video/Chat iFrame based on type ===-->
        <template v-if="cellContent">
            <v-sheet
                rounded="md"
                color="transparent"
                class="cell-content"
                :class="{ 'pa-6 pb-1': pausedMode, 'chat-cell': isChat }"
            >
                <div
                    class="mv-frame ma-auto"
                    :class="{ 'elevation-4': pausedMode }"
                    v-if="cellContent.type === 'video' && cellContent.content.id"
                    :key="'v' + uniqueId"
                >
                    <!-- Twitch Player -->
                    <VueTwitchPlayer
                        v-if="isTwitchVideo"
                        :channel="cellContent.content.id"
                        :playsInline="true"
                        @ready="vidReady"
                        @ended="pausedMode = true"
                        @play="vidPlaying({ data: 1 })"
                        @pause="vidPlaying({ data: 2 })"
                        @error="pausedMode = true"
                        :mute="muted"
                        ref="twitchPlayer"
                    >
                    </VueTwitchPlayer>
                    <!-- Youtube Player -->
                    <youtube
                        v-else
                        :key="'ytplayer-' + item.i + cellContent.content.id"
                        :video-id="cellContent.content.id"
                        :playerVars="{
                            playsinline: 1,
                        }"
                        @ready="vidReady"
                        @ended="pausedMode = true"
                        @playing="vidPlaying"
                        @paused="vidPlaying"
                        @cued="pausedMode = true"
                        @error="pausedMode = true"
                        :mute="muted"
                    >
                        <!--                         @buffering="pausedMode=true" -->
                    </youtube>
                </div>
                <template v-else-if="cellContent.type === 'chat'">
                    <TabbedLiveChat
                        :activeVideos="activeVideos"
                        :setShowTL="toggleTL"
                        :setShowChat="toggleChat"
                        :id="item.i"
                    />
                </template>
            </v-sheet>

            <template v-if="isVideo && pausedMode">
                <!-- VIDEO + PAUSED --->
                <CellControl
                    :playIcon="icons.mdiPlay"
                    @playpause="ytPlayer.playVideo()"
                    @reset="uniqueId = Date.now()"
                    @back="resetCell"
                    @delete="deleteCell"
                    class="ma-6 mt-0"
                />
            </template>
            <template v-if="isChat && pausedMode">
                <!-- CHAT + PAUSED --->
                <CellControl
                    :playIcon="icons.mdiCheck"
                    @back="resetCell"
                    @playpause="pausedMode = !pausedMode"
                    @delete="deleteCell"
                    class="ma-6 mt-0"
                />
            </template>
            <template v-if="isChat && !pausedMode">
                <!-- CHAT + UNPAUSED --->
                <v-sheet class="cell-control">
                    <v-btn :x-small="toggleChat || toggleTL" width="50%" @click="pausedMode = !pausedMode"
                        ><v-icon small>{{ icons.mdiMenu }}</v-icon
                        >{{ $t("component.videoCard.edit") }}</v-btn
                    >
                    <v-btn
                        :x-small="toggleChat || toggleTL"
                        width="25%"
                        @click="toggleChatHandle"
                        :color="toggleChat ? 'primary' : ''"
                    >
                        <v-icon small>{{ icons.mdiTranslate }}</v-icon
                        >Chat
                    </v-btn>
                    <v-btn
                        :x-small="toggleChat || toggleTL"
                        width="25%"
                        @click="toggleTLHandle"
                        :color="toggleTL ? 'primary' : ''"
                    >
                        <v-icon small>{{ icons.mdiTranslate }}</v-icon
                        >TL
                    </v-btn>
                </v-sheet>
                <div style="height: 20%" v-if="!toggleChat && !toggleTL"></div>
            </template>
        </template>
    </v-card>
</template>

<script lang="ts">
import { mdiMessage, mdiArrowLeftCircle } from "@mdi/js";
import TabbedLiveChat from "@/components/multiview/TabbedLiveChat.vue";
import { mapState, mapGetters } from "vuex";
import CellControl from "./CellControl.vue";

export default {
    name: "Cell",
    components: {
        TabbedLiveChat,
        // VideoCardList,
        CellControl,
        VueTwitchPlayer: () => import("./TwitchPlayer.vue"),
    },
    props: {
        item: {
            type: Object,
            required: true,
        },
    },
    data() {
        return {
            mdiMessage,
            mdiArrowLeftCircle,
            pausedMode: true,
            uniqueId: Date.now(),
            ytPlayer: null,
            toggleTL: false,
            toggleChat: true,
        };
    },
    mounted() {
        // initialize chat cell in non paused mode
        if (this.cellContent?.type === "chat") this.pausedMode = false;
        this.setLayoutFreeze();
    },
    watch: {
        cellContent(nw, old) {
            // if cell becomes null or content changes to a different type, set paused mode back to true
            if (!nw || (old && nw && nw.type !== old.type)) this.pausedMode = true;
            if (nw && nw.type === "chat") this.pausedMode = false;
            this.setLayoutFreeze();

            if (
                nw &&
                nw.type === "video" &&
                this.iOS() &&
                this.$store.state.multiview.layout.find((item) => {
                    return (
                        item.i !== this.item.i &&
                        this.layoutContent[item.i] &&
                        this.layoutContent[item.i].type === "video" // &&
                    );
                })
            ) {
                this.muted = true;
            }
        },
        pausedMode(newMode) {
            this.setLayoutFreeze(newMode);
        },
    },
    computed: {
        ...mapGetters("multiview", ["activeVideos"]),
        ...mapState("multiview", ["layoutContent"]),
        cellContent() {
            return this.layoutContent[this.item.i];
        },
        isChat() {
            return this.cellContent.type === "chat";
        },
        isVideo() {
            return this.cellContent.type === "video";
        },
        isTwitchVideo() {
            return (
                this.cellContent &&
                this.cellContent.type === "video" &&
                this.cellContent.content.cellVideoType === "twitch"
            );
        },
        muted: {
            get() {
                if (!this.cellContent) return false;
                return this.cellContent.muted;
            },
            set(value) {
                this.$store.commit("multiview/muteLayoutContent", { id: this.item.i, value });
            },
        },
    },
    methods: {
        // getVideoThumbnails,
        setItemAsChat(item) {
            this.$store.commit("multiview/setLayoutContentById", {
                id: item.i,
                content: {
                    type: "chat",
                },
            });
        },
        deleteCell() {
            this.$emit("delete", this.item.i);
        },
        vidPlaying(evt) {
            this.pausedMode = evt.data === 2;
            if (evt.data === 2 && this.iOS() && this.ytPlayer) {
                this.ytPlayer.mute();
                this.muted = true;
            }
        },
        vidReady(evt) {
            if (evt) this.ytPlayer = evt.target;
        },
        resetCell() {
            this.$store.commit("multiview/deleteLayoutContent", this.item.i);
        },
        toggleChatHandle() {
            this.toggleChat = !this.toggleChat;
            if (!this.toggleChat && !this.toggleTL) this.toggleTL = true;
        },
        toggleTLHandle() {
            this.toggleTL = !this.toggleTL;
            if (!this.toggleChat && !this.toggleTL) this.toggleChat = true;
        },
        iOS() {
            return (
                ["iPad Simulator", "iPhone Simulator", "iPod Simulator", "iPad", "iPhone", "iPod"].includes(
                    navigator.platform,
                ) ||
                // iPad on iOS 13 detection
                (navigator.userAgent.includes("Mac") && "ontouchend" in document)
            );
        },
        setLayoutFreeze(newMode = this.pausedMode) {
            if (newMode) this.$store.commit("multiview/unfreezeLayoutItem", this.item.i);
            else this.$store.commit("multiview/freezeLayoutItem", this.item.i);
        },
    },
};
</script>

<style lang="scss">
.mv-cell {
    background-size: contain;
    background-position: center;
    height: 100%;
    border: 1px solid #f0629118 !important;
    justify-content: flex-start;
    align-content: stretch;
    flex-direction: column;

    .cell-content {
        display: flex;
        flex-wrap: wrap;
        flex-grow: 1;
        flex-basis: 100%;
        flex-shrink: 1;
        max-height: 100%;
        height: 100%;
    }
}

.mv-cell.edit-mode {
    border: 1px solid var(--v-secondary-base) !important;
}

.vue-grid-item.vue-draggable-dragging .mv-cell,
.vue-grid-item.resizing .mv-cell {
    pointer-events: none;
}

.mv-frame > div > iframe {
    position: absolute;
    width: 100%;
    height: 100%;
}

.mv-frame {
    position: relative;
    width: 100%;
    height: 100%;
}
</style>
