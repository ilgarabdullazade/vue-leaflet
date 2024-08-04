<script lang="ts">
import type L from "leaflet";
import "leaflet-polylineoffset";
import {
  defineComponent,
  inject,
  markRaw,
  nextTick,
  onMounted,
  ref,
  watch,
} from "vue";

import { render } from "@src/functions/layer";
import { polylineProps, setupPolyline } from "@src/functions/polyline";
import {
  AddLayerInjection,
  UseGlobalLeafletInjection,
} from "@src/types/injectionKeys";
import {
  WINDOW_OR_GLOBAL,
  assertInject,
  propsBinder,
  remapEvents,
} from "@src/utils";

type PolylineWithOffset = L.Polyline & {
  setOffset: (value: number) => typeof L;
};
/**
 * Polyline component, lets you add and personalize polylines on the map
 */
export default defineComponent({
  name: "LPolylineOffset",
  props: {
    ...polylineProps,
    offset: {
      type: Number,
      default: 0,
    },
  },
  emits: ["ready"],
  setup(props, context) {
    const leafletObject = ref<PolylineWithOffset>();
    const ready = ref(false);

    const useGlobalLeaflet = inject(UseGlobalLeafletInjection);
    const addLayer = assertInject(AddLayerInjection);

    const { options, methods } = setupPolyline(props, leafletObject, context);

    watch(
      () => props.offset,
      (val) => {
        leafletObject.value?.setOffset(val);
      }
    );

    onMounted(async () => {
      const { polyline }: typeof L = useGlobalLeaflet
        ? WINDOW_OR_GLOBAL.L
        : await import("leaflet/dist/leaflet-src.esm");

      leafletObject.value = markRaw<PolylineWithOffset>(
        polyline(props.latLngs, options) as PolylineWithOffset
      );

      const { listeners } = remapEvents(context.attrs);
      leafletObject.value.on(listeners);

      propsBinder(methods, leafletObject.value, props);

      addLayer({
        ...props,
        ...methods,
        leafletObject: leafletObject.value,
      });

      leafletObject.value?.setOffset(props.offset);

      ready.value = true;
      nextTick(() => context.emit("ready", leafletObject.value));
    });

    return { ready, leafletObject };
  },
  render() {
    return render(this.ready, this.$slots);
  },
});
</script>
