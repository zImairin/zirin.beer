<script>
    import { T } from '@threlte/core';
	import { OrbitControls, Sky } from '@threlte/extras';
</script>

<T.PerspectiveCamera makeDefault position={[10, 1, 0]} zoom = {1.5} fov ={70} near={1}>
    <OrbitControls 
    autoRotate
    autoRotateSpeed={0.5}
    enableDamping
    />
</T.PerspectiveCamera>

<Sky />

<T.Mesh position={[0, 1, 0]} scale={1}>
    <T.SphereGeometry/>
    <T.MeshStandardMaterial color="hotpink" roughness={0.3} metalness={0.7}/>
</T.Mesh>

<T.GridHelper />