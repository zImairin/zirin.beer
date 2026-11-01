<script>
    import { T } from '@threlte/core';
	import { OrbitControls, Environment } from '@threlte/extras';
    import FunyShawarma from '../models/funy_shawarma.svelte';
</script>

<T.PerspectiveCamera makeDefault position={[10, 1, 0]} zoom = {1.5} fov ={70} near={1}>
    <OrbitControls 
    // autoRotate
    // autoRotateSpeed={0.5}
    enableDamping
    />
</T.PerspectiveCamera>

<!-- <Sky /> -->

<!-- <T.Mesh position={[0, 1, 0]} scale={1}>
    <T.SphereGeometry/>
    <T.MeshStandardMaterial color="hotpink" roughness={0.3} metalness={0.7}/>
</T.Mesh> -->

<!-- <T.GridHelper /> -->

<T.Group position={[-5, -1.5, -10]} rotation={[0, -Math.PI/3, 0]} scale={7}>
    <FunyShawarma />
</T.Group>

<Environment
    url="/HDR/the_sky_is_on_fire_1k.hdr"
    isBackground={true}
/>