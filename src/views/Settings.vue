<template>
    <v-container>
        <v-row>
            <v-col>
                <div class="text-h4">Settings</div>
                <v-switch
                    v-model="darkMode"
                    label="Dark Mode"
                    messages="Changes the theme between light mode and dark mode"
                ></v-switch>
                <v-switch
                    v-model="redirectMode"
                    label="Redirect Mode"
                    messages="Clicking on a video takes you straight to the youtube video"
                ></v-switch>
                <v-switch
                    v-model="useEnName"
                    label="Use English Name"
                    messages="Replaces Vtuber name to their English name throughout the site"
                ></v-switch>
                <v-switch
                    v-model="hideThumbnail"
                    label="Hide Video Thumbnails"
                    messages="Hide video thumbnails, helps speed up loading time and less scrolling on mobile"
                ></v-switch>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
export default {
    name: "Settings",
    computed: {
        darkMode: {
            get() {
                return this.$store.state.darkMode;
            },
            set(val) {
                this.$store.commit("setDarkMode", val);
            },
        },
        redirectMode: {
            get() {
                return this.$store.state.redirectMode;
            },
            set(val) {
                this.$store.commit("setRedirectMode", val);
            },
        },
        useEnName: {
            get() {
                return this.$store.getters.useEnName;
            },
            set(val) {
                this.$store.commit("setUseEnName", val);
            },
        },
        hideThumbnail: {
            get() {
                return this.$store.state.hideThumbnail;
            },
            set(val) {
                this.$store.commit("setHideThumbnail", val);
            },
        },
    },
};
</script>

<style></style>