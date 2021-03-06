<template>
    <div class="app-container">
        <div v-if="selectedFiles.length > 0">
            <file-details></file-details>
            <package-selection></package-selection>
            <start-button></start-button>
        </div>
        <el-dialog :title="$t('views.apps.simon.analysis.index.dialog.title')" :visible.sync="errorDialogVisible" width="30%" center :close-on-click-modal="false" :close-on-press-escape="false" :show-close="false">
            <span v-html="errorDialogMessage"></span>
            <span slot="footer" class="dialog-footer">
                <el-button @click="cancelInitFeatures">{{ $t('views.apps.simon.analysis.index.dialog.buttons.cancel') }}</el-button>
                <el-button type="primary" @click="reinitializeInitFeatures">{{ $t('views.apps.simon.analysis.index.dialog.buttons.try_again') }}</el-button>
            </span>
        </el-dialog>
    </div>
</template>
<script>
import { createHashFromArrayID } from "@/utils/helpers";
import { FileDetails, PackageSelection, StartButton } from "./components";
import { htmlentities } from "@/utils/3rdparty/htmlentities";

export default {
    name: "layout",
    components: {
        FileDetails,
        PackageSelection,
        StartButton
    },
    data() {
        return {
            loadingInstance: null,
            errorDialogVisible: false,
            errorDialogMessage: ""
        };
    },
    computed: {
        avaliablePackages: {
            get() {
                return this.$store.getters.simonAvaliablePackages;
            },
            set(value) {
                this.$store.dispatch("setSimonAvaliablePackages", value);
            }
        },
        selectedPackages: {
            get() {
                return this.$store.getters.simonSelectedPackages;
            },
            set(value) {
                this.$store.dispatch("setSimonSelectedPackages", value);
            }
        },
        selectedFiles: {
            get() {
                return this.$store.getters.selectedFiles;
            },
            set(value) {
                this.$store.dispatch("setSelectedFiles", value);
            }
        },
        selectedFilesHash: {
            get() {
                return this.$store.getters.simonAnalysisSelectedFileHash;
            },
            set(value) {
                this.$store.dispatch("setSimonAnalysisSelectedFileHash", value);
            }
        }
    },
    mounted() {
        console.log("mounted: analysis");
        this.initFeatures();
    },
    methods: {
        loadingStart() {
            this.loadingInstance = this.$loading({
                lock: true,
                text: "Loading...",
                spinner: "el-icon-loading",
                fullscreen: true,
                customClass: "loading-api",
                background: "rgba(0, 0, 0, 0.7)"
            });
        },
        loadingEnd() {
            if (this.loadingInstance !== null) {
                this.loadingInstance.close();
            }
        },
        cancelInitFeatures() {
            this.errorDialogVisible = false;
            this.errorDialogMessage = "";
            this.$router.push({
                path: "/workspace/index"
            });
        },
        reinitializeInitFeatures() {
            this.errorDialogVisible = false;
            this.errorDialogMessage = "";
            this.initFeatures();
        },
        initPackages() {
            /** Get packages needed in PackageSelection sub-component but only if we didn't got them already */
            // if (this.avaliablePackages.length === 0 && this.selectedPackages.length === 0) {
            this.loadingStart();
            this.$store.dispatch("setSimonAvailablePackages", this.selectedFiles).then(
                response => {
                    this.loadingEnd();
                },
                error => {
                    console.log("Error Response: ", error);
                    this.loadingEnd();
                    this.errorDialogVisible = true;
                    this.errorDialogMessage = this.$t('views.apps.simon.analysis.index.dialog.messages.error_server');
                }
            );
            // }
        },
        initFeatures() {
            this.initPackages();
            const selectedFilesHash = createHashFromArrayID(this.selectedFiles);
            /** Get selected CSV file Details needed in FileDetails sub-component */
            if (this.selectedFilesHash !== selectedFilesHash) {
                console.log("read new features: " + this.selectedFilesHash);

                this.loadingStart();

                this.$store.dispatch("setSimonAnalysisSelectedFileHash", selectedFilesHash);
                /** Get Features for selected file from back-end */
                this.$store.dispatch("setSimonAnalysisFeatures", this.selectedFiles).then(
                    response => {
                        this.loadingEnd();
                        if (Array.isArray(response)) {
                            let sliceEnd = 5;
                            if (response.length < 5) {
                                sliceEnd = response.length;
                            }
                            // Array of invalid features
                            const featuresExample = htmlentities(response.slice(0, sliceEnd).join(", "));

                            this.errorDialogVisible = true;
                            this.errorDialogMessage = this.$t('views.apps.simon.analysis.index.dialog.messages.error_format') + featuresExample;
                        }
                    },
                    error => {
                        console.log("Error Response: ", error);
                        this.loadingEnd();
                        this.selectedFilesHash = "";
                        this.errorDialogVisible = true;
                        this.errorDialogMessage = this.$t('views.apps.simon.analysis.index.dialog.messages.error_read');
                    }
                );
            }
        }
    }
};
</script>
