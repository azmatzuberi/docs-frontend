<template>
  <section
    class="signature-pad column is-half-desktop is-half-tablet is-full-mobile"
  >
    <div id="app">
      <VueSignaturePad
        width="100%"
        height="500px"
        style="margin: 0 auto"
        ref="signaturePad"
      />
      <div class="signature-pad-buttons">
        <b-button @click="save">Save</b-button>
        <b-button @click="undo">Undo</b-button>
      </div>
    </div>
  </section>
</template>

<script>
export default {
  name: "SignaturePad",
  props: ["version"],
  data() {
    return {};
  },
  methods: {
    save() {
      const { isEmpty, data } = this.$refs.signaturePad.saveSignature();
      console.log(isEmpty);
      this.$store.commit("signature/add", data);
    },
    undo() {
      this.$refs.signaturePad.undoSignature();
    },
  },
};
</script>
<style lang="scss">
.signature-pad {
  margin: 0 auto;
  .app {
    @media (min-width: 769px) and (max-width: 1205px) {
      width: 114%;
    }
    div {
      margin: 0 auto;
    }
  }
  .signature-pad-buttons {
    display: flex;
    justify-content: center;
  }
}
</style>
