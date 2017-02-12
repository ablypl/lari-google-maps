<template>
    <div class="google-map-wrapper">
        <div class="control has-adons is-locate-query is-flex">
            <button class="button" v-if="marker" @click.prevent="clearMarker">Usuń marker</button>
            <input type="text" v-model="query" debounce="1000" v-if="search" @input="locate" class="input">
        </div>
        <div class="gMap" ref="map" :style="style"></div>
        <input type="hidden" name="location[lat]" v-model="form.lat" >
        <input type="hidden" name="location[lng]" v-model="form.lng" >
    </div>
</template>

<script type="text/babel">
    import GoogleMapMarker from './GoogleMapMarker.vue'
    export default {
        name: 'GoogleMap',
        // properties defined with component
        props: {
            height: {
                type: Number,
                default: 400
            },
            width: {
                default: false
            },
            center: {
                default: () => ({
                    lat: 51.8338643,
                    lng: 16.5727437
                })
            },
            mark: {
                type: Boolean,
                default: false
            },
            zoom: {
                type: Number,
                default: 15
            },
            search: {
                type: Boolean,
                default: false
            },
            draggable: {
                type: Boolean,
                default: true
            },
            centerondrop: {
                type: Boolean,
                default: true
            },
            prefix: {
                type: String,
                default: ''
            },
            return_center:{
                type: Boolean,
                default: false
            }
        },
        components: {
            GoogleMapMarker
        },
        // component variables
        data: () => ({
            map: '',
            geocoder: '',
            query: '',
            marker: '',
            form: {
                lat: '',
                lng: ''
            }
        }),
        // computed props
        computed: {
            addressQuery(){
                return this.prefix + this.query
            },
            style() {
                let width = '100%';
                let height = `${this.height}px`;
                if(this.width){
                    width = `${this.width}px`;
                }

                return {
                    width,
                    height
                }
            }
        },
        watch: {
            form($value) {
                this.map.setCenter($value)
            }
        },
        // component methods
        methods: {
            initMap() {
                this.map = new this.$google.maps.Map(this.$refs.map, {
                    center: this.center,
                    zoom: this.zoom
                });
            },
            locate() {
                if(!this.search || !this.query){
                    return;
                }
                let component = this;

                this.geocoder = new this.$google.maps.Geocoder();
                this.geocoder.geocode({ address: this.addressQuery }, (results, status) => {
                    if(status !== component.$google.maps.GeocoderStatus.OK){
                        return;
                    }
                    EventBus.$emit('GoogleMapApiFoundLocation', results[0].geometry.location);
                });

                EventBus.$on('GoogleMapApiFoundLocation', (center) => {
                    this.setCenter({
                        lat: center.lat(),
                        lng: center.lng()
                    });
                    if(this.marker){
                        this.marker.setMap(null);
                    }
                    if(!this.mark){
                        return;
                    }

                    this.marker = this.addMarker(center);

                    this.$google.maps.event.addListener(this.marker, "dragend", event => {
                        let location = {
                            lat: event.latLng.lat(),
                            lng: event.latLng.lng()
                        };
                        EventBus.$emit('GoogleMapApiMarkerDropped', location);
                    })

                    EventBus.$on('GoogleMapApiMarkerDropped', center => {
                        if(this.centerondrop){
                            this.setCenter(center)
                        }
                    })
                });
            },
            setCenter(data) {
                this.map.setCenter(data);
                this.form = {
                    lat: data.lat,
                    lng: data.lng
                };
            },
            clearMarker(){
                this.marker.setMap(null);
                this.marker = '';
                this.query = '';
            },
            addMarker(position) {
                return new this.$google.maps.Marker({
                    map: this.map,
                    draggable: this.draggable,
                    position
                });
            }
        },
        // executes when component is created
        beforeMount(){
            EventBus.$on('GoogleMapsApiLoaded', () => {
                this.initMap();

                if(this.mark){
                    this.marker = this.addMarker(this.center);
                }
            })
        }
    }
</script>
<style>
    .google-map-wrapper{
        position: relative;
    }
    .is-locate-query{
        position: absolute;
        right: 5px;
        top: 5px;
        z-index: 10;
    }
    .is-locate-query .input{
        width: 250px
    }
</style>