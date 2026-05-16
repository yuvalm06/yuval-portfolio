# renders/

Drop your Blender PNG renders here.

Naming convention:
  scooter.png
  sprocket.png
  [product-name].png

Each render should be:
- 2680 × 1560px (2× the card at 1340×780)
- Transparent background (PNG)
- Orthographic camera: X=60°, Z=45°
- Off-white clay material (#E8E6E1), HDRI overcast lighting

To use in index.html, replace the scene div:
  <div class="scene">
    <img class="scene-img" src="renders/scooter.png" alt="Scooter" />
  </div>
