<template>
    <div>
        <md-toolbar class="md-primary">
            <div class="md-toolbar-row">
                <img class="logo" src="imgs/rlbot_logo.png">
                <h3 class="md-title" style="flex: 1">RLBot Sandbox</h3>

                <div class="md-toolbar-section-end">
                    <md-button @click="watching = false; $router.replace('/')">
                        Back
                    </md-button>
                </div>
            </div>
        </md-toolbar>
        <div>
            <div class="md-layout">
                <div class="md-layout-item">
                    <md-card class="settings-card">
<!--                        This v-stage thing comes from here: https://konvajs.org/docs/vue/index.html-->
                        <v-stage ref="stage" :config="configKonva" class="arena-background">
                            <v-layer ref="layer">
                                <v-circle :config="ball" @dragend="handleBallDragEnd" @dragstart="handleDragStart"></v-circle>
                                <v-rect v-for="car in cars" :config="car" @dragend="handleCarDragEnd" @dragstart="handleDragStart"></v-rect>
                            </v-layer>
                        </v-stage>
                    </md-card>
                </div>
                <div class="md-layout-item">
                    <md-card class="settings-card">
                        <md-card-content>
                            <md-checkbox v-model="watching">
                                Watch Game
                            </md-checkbox>
                            <md-checkbox v-model="frozen">
                                Freeze Game
                            </md-checkbox>
                            <div>
                                <md-radio v-model="gravity" value="normal">Normal gravity</md-radio>
                                <md-radio v-model="gravity" value="zero">Zero gravity</md-radio>
                            </div>
                            <div>
                                <md-button class="md-raised md-alternative" @click="rewind()"  :disabled="!hasPacketHistory">
                                    Rewind 5 Seconds
                                </md-button>
                            </div>
                            <div class="md-layout md-alignment-center-left">
                                <div class="md-layout-item">
                                    <md-field>
                                        <label>Game Speed</label>
                                        <md-input v-model="gamespeed" type="number"></md-input>
                                    </md-field>
                                </div>
                                <div class="md-layout-item">
                                    <md-button class="md-raised md-alternative" @click="setGamespeed()">Set</md-button>
                                </div>
                            </div>
                            <div class="md-layout md-alignment-center-left">
                                <div class="md-layout-item">
                                    <md-field>
                                        <label>Console Command</label>
                                        <md-input v-model="command"></md-input>
                                    </md-field>
                                </div>
                                <div class="md-layout-item">
                                    <md-button class="md-raised md-alternative" @click="executeCommand()">Execute</md-button>
                                </div>
                            </div>
                            <p>
                                You can drag and drop to move objects around in the game!
                            </p>
                        </md-card-content>
                    </md-card>
                </div>
            </div>
        </div>
    </div>
</template>
<style>
    .arena-background {
        background: url('imgs/arena_diagram.png');
        background-size: contain;
        background-repeat: no-repeat;
        margin: 10px;
    }
</style>
<script>

    // https://konvajs.org/docs/vue/index.html
    Vue.use(VueKonva);

    const PIXEL_HEIGHT = 580;
    const HISTORY_SECONDS = 5;
    const HISTORY_INCREMENT_SECONDS = 0.1;

    let packetHistory = [];

    module.exports = {
        name: 'sandbox',
        data() {
            return {
                watching: false,
                frozen: false,
                gravity: 'normal',
                command: null,
                gamespeed: 1,
                gameTickPacket: null,
                configKonva: {
                    width: 410,
                    height: PIXEL_HEIGHT
                },
                ball: {
                    x: 100,
                    y: 100,
                    radius: 10,
                    fill: 'gray',
                    stroke: 'black',
                    strokeWidth: 2,
                    draggable: true
                },
                cars: [],
                hasPacketHistory: false
            };
        },
        methods: {
            startWatching: function () {
                eel.fetch_game_tick_packet_json()(this.gameTickPacketReceived)
            },
            gameTickPacketReceived: function (result) {
                this.gameTickPacket = result;
                if (!this.dragging) {
                    const ballLoc = this.toCanvasVec(result.game_ball.physics.location);
                    this.ball.x = ballLoc.x;
                    this.ball.y = ballLoc.y;
                    this.ball.z = ballLoc.z;
                    this.cars = [];
                    for (let i = 0; i < result.game_cars.length; i++) {

                        car = {
                            draggable: true,
                            fill: result.game_cars[i].team === 0 ? 'blue' : 'orange',
                            width: 16,
                            height: 10,
                            offset: {
                                x: 8,
                                y: 5
                            },
                            playerIndex: i
                        };

                        const phys = result.game_cars[i].physics;
                        const carLoc = this.toCanvasVec(phys.location);
                        car.x = carLoc.x;
                        car.y = carLoc.y;
                        car.rotation = phys.rotation.yaw * 180 / Math.PI;
                        this.cars.push(car);
                    }
                }

                if (packetHistory.length) {
                    const tail = packetHistory[packetHistory.length - 1];
                    const tailTime = tail.game_info.seconds_elapsed;
                    if (result.game_info.seconds_elapsed - tailTime > HISTORY_INCREMENT_SECONDS) {
                        packetHistory.push(result);
                        if (packetHistory.length > HISTORY_SECONDS / HISTORY_INCREMENT_SECONDS) {
                            packetHistory.shift();
                        }
                    }
                } else {
                    packetHistory.push(result);
                }
                this.hasPacketHistory = true;

                if (this.watching) {
                    setTimeout(function() {
                        eel.fetch_game_tick_packet_json()(this.gameTickPacketReceived)
                    }.bind(this), 50);  // Delay 50 milliseconds to avoid hurting the CPU
                }
            },
            toCanvasVec: function(packetVec) {
                // Height without goals: 512 px
                // Total width: 410 px
                return {x: packetVec.x / -20 + 205, y: packetVec.y / -20 + PIXEL_HEIGHT / 2, z: packetVec.z / 20};
            },
            toPacketVec: function(canvasVec) {
                return {x: (canvasVec.x - 205) * -20, y: (canvasVec.y - PIXEL_HEIGHT / 2) * -20, z: canvasVec.z * 20}
            },
            handleDragStart: function (evt) {
                this.dragging = true;
            },
            handleBallDragEnd: function (evt) {
                this.dragging = false;
                const packetVec = this.toPacketVec({x: evt.target.x(), y: evt.target.y(), z: 10});
                eel.set_state({
                    ball:{physics:{location:packetVec, velocity: {x: 0, y: 0, z: 0}}}
                });
            },
            handleCarDragEnd: function (evt) {
                const index = evt.target.attrs.playerIndex;
                this.dragging = false;
                const packetVec = this.toPacketVec({x: evt.target.x(), y: evt.target.y(), z: 10});
                cars = {};
                cars[index] = {physics:{location:packetVec, velocity: {x: 0, y: 0, z: 0}}};
                eel.set_state({
                    cars: cars
                });
            },
            executeCommand: function() {
                eel.set_state({console_commands: [this.command]});
            },
            setGamespeed: function() {
                eel.set_state({game_info: {game_speed: parseFloat(this.gamespeed)}});
            },
            rewind: function () {
                // Rewinds to the earliest point in recorded history. In many cases this will be 5 seconds ago,
                // assuming HISTORY_SECONDS is still configured as 5. However, due to low amount of history data or
                // interruptions in recording, this can vary.

                // The rewinding is not perfect because state setting cannot actually change the clock or change
                // the state of boost pads. Also state setting is a little mushy in general, so consider this to be
                // "best effort".

                if (packetHistory.length > 0) {
                    firstPacket = packetHistory[0];
                    lastPacket = packetHistory[packetHistory.length - 1];

                    // Doctor the time on the first packet. It will remain in the packet history array.
                    // With the new time, it will be representative of what's about to happen due to state setting.
                    // This will prevent any "bad history" from leaking in due to latency before the state setting takes effect.
                    firstPacket.game_info.seconds_elapsed = lastPacket.game_info.seconds_elapsed;

                    // Truncate the packet history
                    packetHistory.length = 1;

                    cars = {};
                    firstPacket.game_cars.forEach(function (car, index) {
                        cars[index] = {physics: car.physics, boost_amount: car.boost}
                    });
                    eel.set_state({
                        cars: cars,
                        ball: {physics: firstPacket.game_ball.physics}
                    });
                }
                this.hasPacketHistory = false;
            }
        },
        watch: {
            watching: {
                handler: function (newVal) {
                    if (newVal) {
                        this.startWatching();
                    }
                }
            },
            frozen: {
                handler: function (newVal) {
                    eel.set_state({game_info: {paused: newVal}});
                }
            },
            gravity: {
                handler: function (newVal) {
                    if (newVal === 'zero') {
                        eel.set_state({game_info: {world_gravity_z: -0.000001}});
                    } else {
                        eel.set_state({game_info: {world_gravity_z: -650}});
                    }
                }
            }
        }
    };

</script>
