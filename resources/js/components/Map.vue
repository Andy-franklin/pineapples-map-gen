<template>
    <div>
        <div v-if="!loading && !hasRun">
            <b-button variant="outline-primary" @click="run()" class="mb-2">Run</b-button>
            <br/>Or<br/>
            <b-button variant="outline-secondary" class="mt-2" v-b-modal.import_modal>Import from file</b-button>

            <b-modal id="import_modal" title="Import from file" @ok="importFromFile">
                <p>Your file must be a .pineapples-map file to import correctly.</p>
                <b-form-file v-model="importFile" ref="file-input" class="mb-2"></b-form-file>
            </b-modal>
        </div>

        <div v-if="loading">
            <div class="lds-facebook"><div></div><div></div><div></div></div>
            <div><p>{{ loadingText }}</p></div>
        </div>

        <div v-show="!loading && hasRun">
            <canvas id="map" :width="canvasWidth" :height="canvasWidth" class="m-b-md" style="width: 600px; height: 600px"></canvas>
            <br/>
            <p>Continue map from:
                <a href="#">top</a> |
                <a href="#">left</a> |
                <a href="#">right</a> |
                <a href="#">bottom</a>
            </p>

            <b-button variant="outline-primary" @click="run()" class="mr-4">Run Again</b-button>
            <b-button @click="exportToFile">Export to file</b-button>
        </div>
    </div>
</template>

<script>
    //Using the Diamond Square Algorithm
    //https://en.wikipedia.org/wiki/Diamond-square_algorithm
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

                dimension: 1024,
                canvasWidth: 1024,

                unitSize: 1,
                roughness: 8,

                importFile: [],
            }
        },
        mounted() {
        },
        methods: {
            makeToast: function(content, title, variant = null) {
                this.$bvToast.toast(content, {
                    title: title,
                    variant: variant,
                    solid: true
                })
            },
            importFromFile: function(bvModalEvt) {
                bvModalEvt.preventDefault();

                const reader = new FileReader();

                reader.readAsText(this.importFile);
                reader.onload = () => {
                    this.map = JSON.parse(atob(reader.result));
                    this.$bvModal.hide('modal-prevent-closing');

                    this.drawMap();

                    // this.makeToast('Your file is invalid.', 'Error', 'danger')
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
            exportToFile: function() {
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

                this.download(name + '.pineapples-map', exportString)
            },
            run: function() {
                this.loading = true;
                this.hasRun = true;

                this.$worker.run((data) => {
                    for (let x = 0; x < data.dimension + 1; x ++) { //width
                        data.map[x] = [];
                        for (let y = 0; y < data.dimension + 1; y ++) { //height
                            if (50 > Math.floor(Math.random() * data.dimension)) {
                                data.map[x][y] = 1;
                            } else {
                                data.map[x][y] = 0;
                            }
                        }
                    }

                    function seed() {
                        data.map[0][0] = Math.random();
                        data.topLeft = data.map[0][0];

                        // bottom left
                        data.map[0][data.dimension] = Math.random();
                        data.bottomLeft = data.map[0][data.dimension];

                        // top right
                        data.map[data.dimension][0] = Math.random();
                        data.topRight = data.map[data.dimension][0];

                        // bottom right
                        data.map[data.dimension][data.dimension] = Math.random();
                        data.bottomRight = data.map[data.dimension][data.dimension];

                        //middle
                        data.map[data.dimension / 2][data.dimension / 2] = data.map[0][0] + data.map[0][data.dimension] + data.map[data.dimension][0] + data.map[data.dimension][data.dimension] / 4;
                        data.map[data.dimension / 2][data.dimension / 2] = normalize(data.map[data.dimension / 2][data.dimension / 2]);
                        data.center = data.map[data.dimension / 2][data.dimension / 2];

                        //Start displacement
                        midpointDisplacement(data.dimension)
                    }
                    function midpointDisplacement(dimension) {
                        //Half the original dimension
                        let newDimension = 2 * Math.round(dimension / 2) - 2;

                        //If we are now smaller than the unitsize then we are done
                        if (newDimension <= data.unitSize) return;

                        for (let i = newDimension; i <= data.dimension; i += newDimension) {
                            for (let j = newDimension; j <= data.dimension; j += newDimension) {
                                let x = i - (newDimension / 2);
                                let y = j - (newDimension / 2);

                                let topLeft = data.map[i - newDimension][j - newDimension];
                                let topRight = data.map[i][j - newDimension];
                                let bottomLeft = data.map[i - newDimension][j];
                                let bottomRight = data.map[i][j];

                                // Center
                                data.map[x][y] = (topLeft + topRight + bottomLeft + bottomRight) / 4 + displace(dimension);
                                data.map[x][y] = normalize(data.map[x][y]);
                                let center = data.map[x][y];

                                // Top
                                if (j - (newDimension * 2) + (newDimension / 2) > 0) {
                                    data.map[x][j - newDimension] = (topLeft + topRight + center + data.map[x][j - dimension + (newDimension / 2)]) / 4 + displace(dimension);
                                } else {
                                    data.map[x][j - newDimension] = (topLeft + topRight + center) / 3 + displace(dimension);
                                }

                                data.map[x][j - newDimension] = normalize(data.map[x][j - newDimension]);

                                // Bottom
                                if (j + (newDimension / 2) < data.dimension) {
                                    data.map[x][j] = (bottomLeft + bottomRight + center + data.map[x][j + (newDimension / 2)]) / 4 + displace(dimension);
                                } else {
                                    data.map[x][j] = (bottomLeft + bottomRight + center) / 3 + displace(dimension);
                                }

                                data.map[x][j] = normalize(data.map[x][j]);

                                //Right
                                if (i + (newDimension / 2) < data.dimension) {
                                    data.map[i][y] = (topRight + bottomRight + center + data.map[i + (newDimension / 2)][y]) / 4 + displace(dimension);
                                } else {
                                    data.map[i][y] = (topRight + bottomRight + center) / 3 + displace(dimension);
                                }

                                data.map[i][y] = normalize(data.map[i][y]);

                                // Left
                                if (i - (newDimension * 2) + (newDimension / 2) > 0) {
                                    data.map[i - newDimension][y] = (topLeft + bottomLeft + center + data.map[i - dimension + (newDimension / 2)][y]) / 4 + displace(dimension);
                                } else {
                                    data.map[i - newDimension][y] = (topLeft + bottomLeft + center) / 3 + displace(dimension);
                                }

                                data.map[i - newDimension][y] = normalize(data.map[i - newDimension][y]);
                            }
                        }

                        midpointDisplacement(newDimension);
                    }
                    function normalize(value) {
                        return Math.max(Math.min(value, 1), 0);
                    }
                    function displace(number) {
                        let max = number / (data.dimension + data.dimension) * data.roughness;
                        let val = (Math.random() - 0.5) * max;

                        return val;
                    }

                    seed();
                    return data.map;
                }, [this._data])
                    .then(result => {
                        this.map = result;
                        this.drawMap();
                    })
                    .catch(e => {
                        console.error(e)
                    });
            },
            drawMap: function() {
                this.loading = true;
                this.loadingText = 'Colouring Map';
                this.$worker.run((data) => {
                    function fade(startColor, endColor, steps, step){
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
                    let waterStart={r:10,g:20,b:40},
                        waterEnd={r:39,g:50,b:63},
                        grassStart={r:22,g:38,b:3},
                        grassEnd={r:67,g:100,b:18},
                        mtnEnd={r:60,g:56,b:31},
                        mtnStart={r:67,g:80,b:18},
                        rocamtStart={r:90,g:90,b:90},
                        rocamtEnd={r:130,g:130,b:130},
                        snowStart={r:255,g:255,b:255},
                        snowEnd={r:200,g:200,b:200};

                    for (let x = 0; x <= data.dimension - 1; x += data.unitSize) {
                        for (let y = 0; y <= data.dimension - 1; y += data.unitSize) {
                            let colour = {r: 0, g: 0, b: 0};
                            let  dataPoint = data.map[x][y];
                            
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
                                    let pData = ( ~~(x + w) + ( ~~(y + h) * data.canvasWidth)) * 4;

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
            renderMap: function() {
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