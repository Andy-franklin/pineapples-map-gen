<template>
    <div>
        <div v-if="!loading && !hasRun">
            <b-button variant="outline-primary" class="mb-2" v-b-modal.run_modal>Run</b-button>
            <br/>Or<br/>
            <b-button variant="outline-secondary" class="mt-2" v-b-modal.import_modal>Import from file</b-button>

            <b-modal id="import_modal" title="Import from file" @ok="importFromFile">
                <p>Your file must be a .amby-map file to import correctly.</p>
                <b-form-file v-model="importFile" ref="file-input" class="mb-2"></b-form-file>
            </b-modal>
        </div>

        <b-modal id="run_modal" title="Settings" @ok="run">
            <!--Landmarks-->
            <b-form-group label="Landmarks:">
                <b-form-checkbox-group id="landmarks-group" v-model="landmarks" name="landmarks">
                    <b-form-checkbox value="cities">Cities</b-form-checkbox>
                    <b-form-checkbox value="caves">Caves</b-form-checkbox>
                    <b-form-checkbox value="towns">Towns</b-form-checkbox>
                    <b-form-checkbox value="farms">Farms</b-form-checkbox>
                    <b-form-checkbox value="ruins">Ruins</b-form-checkbox>
                </b-form-checkbox-group>
            </b-form-group>


            <!--Temperature-->
            <b-form-group :label="'Temperature: ' + temperature + String.fromCharCode(176) + 'C'">
                <b-input-group prepend="-20" append="60">
                    <b-form-input type="range" min="-20" max="60" v-model="temperature"></b-form-input>
                </b-input-group>
            </b-form-group>

            <!--Roughness-->
            <b-form-group :label="'Roughness: ' + roughness">
                <b-input-group prepend="0" append="15">
                    <b-form-input type="range" min="0" max="15" v-model="roughness"></b-form-input>
                </b-input-group>
            </b-form-group>
        </b-modal>

        <div v-if="loading">
            <div class="lds-facebook">
                <div></div>
                <div></div>
                <div></div>
            </div>
            <div><p>{{ loadingText }}</p></div>
        </div>

        <div v-show="!loading && hasRun">
            <canvas id="map" :width="canvasWidth" :height="canvasWidth" class="m-b-md"
                    style="width: 600px; height: 600px"></canvas>
            <br/>
            <p>Continue map from:
                <a href="#">top</a> |
                <a href="#">left</a> |
                <a href="#">right</a> |
                <a href="#">bottom</a>
            </p>

            <b-button variant="outline-primary" v-b-modal.run_modal class="mr-4">Run Again</b-button>
            <b-button @click="exportToFile">Export to file</b-button>
        </div>
    </div>
</template>

<script>

    let _simplexNoise = require('simplex-noise');
    let _simplex = new _simplexNoise('123456');
    export default {
        name: "Map",
        data() {
            return {
                loadingText: 'Seeding',
                loading: false,
                hasRun: false,
                map: [],
                imageMap: [],
                topLeft: [],
                bottomLeft: [],
                topRight: [],
                bottomRight: [],
                center: [],
                // dimension: 1024 * 3,
                // canvasWidth: 1024 * 3,

                chunkSize: 1024,
                canvasWidth: 1024,

                unitSize: 1,
                roughness: 8,

                importFile: [],

                landmarks: [],
                temperature: 18,
            }
        },
        mounted() {
        },
        methods: {
            makeToast: function (content, title, variant = null) {
                this.$bvToast.toast(content, {
                    title: title,
                    variant: variant,
                    solid: true
                })
            },
            importFromFile: function (bvModalEvt) {
                bvModalEvt.preventDefault();

                const reader = new FileReader();

                reader.readAsText(this.importFile);
                reader.onload = () => {
                    try {
                        this.map = JSON.parse(atob(reader.result));
                        this.$bvModal.hide('import_modal');
                        this.drawMap();
                    } catch (error) {
                        this.makeToast('Your file is invalid.', 'Error', 'danger')
                    }
                };
            },
            download: function (filename, text) {
                let element = document.createElement('a');
                element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
                element.setAttribute('download', filename);

                element.style.display = 'none';
                document.body.appendChild(element);

                element.click();

                document.body.removeChild(element);
            },
            exportToFile: function () {
                let exportString = btoa(JSON.stringify(this.map));

                const today = new Date();
                const y = today.getFullYear();
                // JavaScript months are 0-based.
                const m = today.getMonth() + 1;
                const d = today.getDate();
                const h = today.getHours();
                const mi = today.getMinutes();
                const s = today.getSeconds();
                let name = y + "-" + m + d + h + mi + s;

                this.download(name + '.amby-map', exportString)
            },
            run: function () {
                this.loading = true;
                this.hasRun = true;

                this.$worker.run((data, simplex) => {
                    function GetChunk(chunkX, chunkY) {
                        let chunk = [];
                        for (let x = 0; x < data.chunkSize; x++) {
                            chunk[x] = [];
                            for (let y = 0; y < data.chunkSize; y++) {
                                chunk[x][y] = GetNoise(x + (chunkX * data.chunkSize), y + (chunkY * data.chunkSize));
                            }
                        }
                        return chunk;
                    }

                    function GetNoise(x, y) {
                        let noise = simplex.noise2D(x, y);
                        let normalised = noise * 0.5 + 0.5;
                        return normalised;
                    }

                    data.map = GetChunk(0, 0);

                    return data.map;
                }, [this._data,_simplex])
                    .then(result => {
                        this.map = result;
                        this.drawMap();
                    })
                    .catch(e => {
                        console.error(e)
                    });
            },
            drawMap: function () {
                this.loading = true;
                this.loadingText = 'Colouring Map';
                this.$worker.run((data) => {
                    function fade(startColor, endColor, steps, step) {
                        let scale = step / steps,
                            r = startColor.r + scale * (endColor.r - startColor.r),
                            b = startColor.b + scale * (endColor.b - startColor.b),
                            g = startColor.g + scale * (endColor.g - startColor.g);

                        return {
                            r: r,
                            g: g,
                            b: b
                        }
                    }

                    let imageMap = [];

                    //colour map
                    let waterStart = {r: 10, g: 20, b: 40},
                        waterEnd = {r: 39, g: 50, b: 63},
                        grassStart = {r: 22, g: 38, b: 3},
                        grassEnd = {r: 67, g: 100, b: 18},
                        mtnEnd = {r: 60, g: 56, b: 31},
                        mtnStart = {r: 67, g: 80, b: 18},
                        rocamtStart = {r: 90, g: 90, b: 90},
                        rocamtEnd = {r: 130, g: 130, b: 130},
                        snowStart = {r: 255, g: 255, b: 255},
                        snowEnd = {r: 200, g: 200, b: 200};

                    for (let x = 0; x <= data.dimension - 1; x += data.unitSize) {
                        for (let y = 0; y <= data.dimension - 1; y += data.unitSize) {
                            let colour = {r: 0, g: 0, b: 0};
                            let dataPoint = data.map[x][y];

                            if (data.temperature < -5) {
                                //Include sea ice
                            }

                            if (data.temperature > 40) {
                                grassStart = {r: 67, g: 100, b: 18};
                                grassEnd = {r: 255, g: 228, b: 181};

                                snowStart = rocamtStart;
                                snowEnd = rocamtEnd;
                            }


                            if (dataPoint >= 0 && dataPoint <= 0.15) {
                                colour = fade(waterStart, waterStart, 15, parseInt(dataPoint * 100, 10) - 5);
                            } else if (dataPoint >= 0.15 && dataPoint <= 0.3) {
                                colour = fade(waterStart, waterEnd, 15, parseInt(dataPoint * 100, 10) - 15);
                            } else if (dataPoint > 0.3 && dataPoint <= 0.7) {
                                colour = fade(grassStart, grassEnd, 45, parseInt(dataPoint * 100, 10) - 30);
                            } else if (dataPoint > 0.7 && dataPoint <= 0.85) {
                                colour = fade(mtnStart, mtnEnd, 15, parseInt(dataPoint * 100, 10) - 70);
                            } else if (dataPoint > 0.85 && dataPoint <= 0.9) {
                                colour = fade(rocamtStart, rocamtEnd, 5, parseInt(dataPoint * 100, 10) - 85);
                            } else if (dataPoint > 0.9 && dataPoint <= 1) {
                                colour = fade(snowStart, snowEnd, 10, parseInt(dataPoint * 100, 10) - 90);
                            }

                            for (let w = 0; w <= data.unitSize; w++) {
                                for (let h = 0; h <= data.unitSize; h++) {
                                    let pData = (~~(x + w) + (~~(y + h) * data.canvasWidth)) * 4;

                                    imageMap[pData] = colour.r;
                                    imageMap[pData + 1] = colour.g;
                                    imageMap[pData + 2] = colour.b;
                                    imageMap[pData + 3] = 255;
                                }
                            }
                        }
                    }

                    return imageMap;
                }, [this._data])
                    .then(result => {
                        this.imageMap = result;
                        this.renderMap();
                    })
                    .catch(e => {
                        console.error(e)
                    });
            },
            renderMap: function () {
                this.loadingText = "Rendering map";
                let canvas = document.getElementById('map');
                let ctx = canvas.getContext("2d");


                let image = ctx.createImageData(canvas.height, canvas.width);

                this.$worker.run((image, imageMap) => {
                    for (let i = 0; i < image.data.length; i++) {
                        image.data[i] = imageMap[i];
                    }

                    return image;
                }, [image, this.imageMap])
                    .then(result => {
                        ctx.putImageData(result, 0, 0);
                        this.loading = false;
                        this.hasRun = true;
                    })
                    .catch(e => {
                        console.error(e)
                    });
            }
        }
    }
</script>

<style scoped>
    .lds-facebook {
        display: inline-block;
        position: relative;
        width: 80px;
        height: 80px;
    }

    .lds-facebook div {
        display: inline-block;
        position: absolute;
        left: 8px;
        width: 16px;
        background: #1d68a7;
        animation: lds-facebook 1.2s cubic-bezier(0, 0.5, 0.5, 1) infinite;
    }

    .lds-facebook div:nth-child(1) {
        left: 8px;
        animation-delay: -0.24s;
    }

    .lds-facebook div:nth-child(2) {
        left: 32px;
        animation-delay: -0.12s;
    }

    .lds-facebook div:nth-child(3) {
        left: 56px;
        animation-delay: 0;
    }

    @keyframes lds-facebook {
        0% {
            top: 8px;
            height: 64px;
        }
        50%, 100% {
            top: 24px;
            height: 32px;
        }
    }
</style>
