<template>

  <KPageContainer class="block">
    <KGrid>
      <KGridItem sizes="100, 50, 50" percentage>
        <h2>
          <slot name="title"></slot>
        </h2>
      </KGridItem>
      <KGridItem sizes="100, 50, 50" percentage alignment="right">
        <KRouterLink
          appearance="flat-button"
          :text="allLinkText"
          :to="allLinkRoute"
          class="btn"
        />
      </KGridItem>
    </KGrid>
    <slot></slot>
  </KPageContainer>

</template>


<script>

  import commonCoach from '../../common';

  export default {
    name: 'Block',
    mixins: [commonCoach],
    props: {
      allLinkText: {
        type: String,
        required: true,
      },
      allLinkRoute: {
        type: Object,
        required: true,
      },
    },
  };

</script>


<style lang="scss" scoped>

  .block {
    margin-top: 16px;
  }

  .btn {
    position: relative;
    top: 4px;
    right: -24px;
  }

</style>
