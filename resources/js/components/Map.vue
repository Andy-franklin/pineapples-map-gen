<template>
    <div>
        <div class="lds-facebook" v-if="loading"><div></div><div></div><div></div></div>
        <canvas id="map" width="512" height="512" class="m-b-md" v-else></canvas>
    </div>
</template>

<script>
    //Using the Diamond Square Algorithm
    //https://en.wikipedia.org/wiki/Diamond-square_algorithm
    export default {
        name: "Map",
        data() {
            return {
                loading: true,
                map: [],
                topLeft: [],
                bottomLeft: [],
                topRight: [],
                bottomRight: [],
                center: [],
                dimension: 512,
                unitSize: 1,
                roughness: 5,
            }
        },
        mounted() {
            this.map = [];

            for (let x = 0; x < this.dimension + 1; x ++) { //width
                this.map[x] = [];
                for (let y = 0; y < this.dimension + 1; y ++) { //height
                    this.map[x][y] = 0;
                }
            }

            this.$worker.run((data) => {
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
                    let newDimension = dimension / 2;

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
                    return (Math.random() - 0.5) * max;
                }

                seed();
                return data.map;
            }, [this._data])
                .then(result => {
                    this.map = result;
                    this.loading = false;
                })
                .catch(e => {
                    console.error(e)
                });
        },
        methods: {

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