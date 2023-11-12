<script lang="ts">
    import '@skeletonlabs/skeleton/themes/theme-rocket.css';
    import '@skeletonlabs/skeleton/styles/skeleton.css';
    import '../app.postcss';
    import './styles.scss';
    import { AppShell } from '@skeletonlabs/skeleton';
</script>

<AppShell>
    <slot />
</AppShell>
