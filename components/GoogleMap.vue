<template>
    <div class="google-map-wrapper">
        <slot></slot>
        <div class="control has-adons is-locate-query is-flex">
            <button class="button" v-if="markers && editMode" @click.prevent="clearMarkers">Usuń marker</button>
            <input type="text" v-model="query" debounce="1000" v-if="search" @input="locate" class="input">
        </div>
        <div class="gMap" ref="map" :style="style"></div>
        <input type="hidden" name="location[lat]" v-model="form.lat" >
        <input type="hidden" name="location[lng]" v-model="form.lng" >
    </div>
</template>

<script type="text/babel">
    export default {
        name: 'GoogleMap',
        // properties defined with component
        props: {
            height: {
                type: Number,
            },
            width: {
                default: false
            },
            offset: {
                type: Number
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
            editMode: {
                type: Boolean,
                default: true
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

        // component variables
        data: () => ({
            map: '',
            geocoder: '',
            query: '',
            markers: [],
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
                let height = `${window.innerHeight - this.offset}px`;
                if(this.width){
                    width = `${this.width}px`;
                }
                if(this.height){
                    height = `${this.height}px`;
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
            },

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
                    if(this.markers){
                        this.clearMarkers()
                    }
                    if(!this.mark){
                        return;
                    }

                    this.markers.push(this.addMarker(center));

                    this.$google.maps.event.addListener(this.markers, "dragend", event => {
                        let location = {
                            lat: event.latLng.lat(),
                            lng: event.latLng.lng()
                        };
                        EventBus.$emit('GoogleMapApiMarkerDropped', location);
                    });


                });
            },
            setCenter(data) {
                this.map.setCenter(data);
            },
            setForm(center) {
                this.form = {
                    lat: center.lat,
                    lng: center.lng
                };
            },
            clearMarker(){
                this.markers.setMap(null);
                this.marker = '';
                this.query = '';
            },
            clearMarkers(){
                this.markers.forEach(marker => {
                    marker.setMap(null);
                });
                this.markers = [];
            },
            addMarker(position) {
                let marker = new this.$google.maps.Marker({
                    map: this.map,
                    draggable: this.draggable,
                    position
                });
                EventBus.$emit('GoogleMapApiMarkerAdded', marker);

                return marker;
            },


            addMarkersFromChildren() {
                EventBus.$on('GoogleMapsApiLoaded', () => {
                    this.$children.forEach(item => {
                        let options = Object.assign({
                                    map: this.map
                                },
                                item.$options.propsData,
                                item.$options.computed
                        );

                        let marker = new this.$google.maps.Marker(options);
                        marker.addListener('click', m => {
                            EventBus.$emit('GoogleMapsMarkerClicked', marker, item);
                        });

                        this.markers.push(marker);
                        EventBus.$emit('GoogleMapApiMarkerAdded', marker);
                    })
                });
                EventBus.$on('GoogleMapsMarkerClicked', (marker, component) => {

                })
            },
            addListeners() {
                if(!this.editMode){
                    return;
                }
                EventBus.$on('GoogleMapApiMarkerAdded', (marker) => {
                    this.$google.maps.event.addListener(marker, "dragend", event => {
                        let location = {
                            lat: event.latLng.lat(),
                            lng: event.latLng.lng()
                        };
                        EventBus.$emit('GoogleMapApiMarkerDropped', location);
                    });
                });
                EventBus.$on('GoogleMapApiMarkerDropped', center => {
                    this.setForm(center);
                    if(this.centerondrop){
                        this.setCenter(center)
                    }
                })
            }
        },
        // executes when component is created
        beforeMount(){
            EventBus.$on('GoogleMapsApiLoaded', () => {
                this.initMap();

                if(this.mark){
                    this.markers.push(
                            this.addMarker(this.center)
                    );
                }
                this.form = this.center;
            });
        },
        mounted(){
            this.addListeners();
            this.addMarkersFromChildren();
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
