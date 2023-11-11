<script lang="ts">
    import '@skeletonlabs/skeleton/themes/theme-rocket.css';
    import '@skeletonlabs/skeleton/styles/skeleton.css';
    import '../app.postcss';
    import './styles.scss';
    import { AppShell, AppBar } from '@skeletonlabs/skeleton';
</script>

<AppShell>
    <!--    <svelte:fragment slot="header">-->
    <!--        <AppBar>-->
    <!--            <svelte:fragment slot="lead">-->
    <!--                <strong class="text-xl uppercase">Invoice Generator</strong>-->
    <!--            </svelte:fragment>-->
    <!--        </AppBar>-->
    <!--    </svelte:fragment>-->

    <slot />
</AppShell>
