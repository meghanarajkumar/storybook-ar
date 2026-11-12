# Storybook AR

A mobile web page that opens the rear camera and repaints the live view with a neural style-transfer network. No app, no login.

## What is in this folder

| File | Purpose |
| --- | --- |
| `index.html` | The whole app: landing page, camera view, painter, tracking, capture |
| `models/style`, `models/transformer` | Neural style-transfer weights (about 12 MB), served from your own host |
| `style-reference.jpg` | **You add this.** The illustration the painter learns its look from |

## Add the reference illustration

Save the reference image next to `index.html` as `style-reference.jpg` (`.png` and `.webp` also work). It is centre-cropped to a square, so choose a crop that shows the brushwork and palette you want. Without it the app falls back to a built-in gouache swatch, which is a stand-in and not the intended look.

## Try it

The camera only works over HTTPS (or on `localhost`).

```
npx serve .                       # then open http://localhost:3000 on this computer
npx localtunnel --port 3000       # gives an https address you can open on a phone
```

## Deploy

It is a static site. Upload the folder as it is to any HTTPS host:

- Netlify: drag the folder onto app.netlify.com/drop
- Vercel: `npx vercel --prod` inside the folder
- Cloudflare Pages or GitHub Pages: publish the folder as the site root

Keep `models/` beside `index.html`. The files inside have no extension on purpose.

## QR code

Open the deployed address on a computer: the landing page shows a QR code for its own address. For print, put the deployed address into any QR generator.

## Options

Add these to the address:

- `?strength=0.5` how far colours move toward the reference, 0 to 1. Lower keeps each object closer to its own colour (default 1).
- `?backend=webgpu` run the network on WebGPU instead of WebGL. Untested on phones, so it is opt-in.
- `?stylizer=https://your-server/paint` send frames to your own model instead of the built-in one (see below).

Other settings (resolution tiers, smoothing, mask rate) are in the `CONFIG` block at the top of the script.

## How it works

1. **Camera and compositor.** `getUserMedia` feeds a Three.js full-screen shader.
2. **Painter.** Each camera frame goes through Magenta's arbitrary style-transfer network in TensorFlow.js. The whole frame is repainted, so objects keep their shape, place and perspective.
3. **People.** MediaPipe's DeepLab v3 segmenter marks the pixels of its "person" class about six times a second and those pixels show the untouched camera. Nothing identifies, tracks or stylises a person.
4. **Steadiness.** The network is slower than the camera, so the app measures how the view has shifted since each painted frame and slides the painting to stay on the scene, cross-fades to each new frame, and blends a share of the previous one in to calm flicker. Resolution steps up or down with the phone's speed.

The shader only positions the network's output, masks people and adds a faint paper grain. If the network fails to load, the app shows the plain camera and says so.

## What this is not yet

- **It is style transfer, not re-illustration.** The network repaints colour, texture and brushwork. It does not redraw a chair the way an illustrator would, with simplified storybook geometry. That needs an image-to-image diffusion model, which cannot run in a phone browser at camera speed today.
- **The stronger model is the missing piece.** `RemoteStylizer` in `index.html` is the socket for it: it POSTs each frame as `image/jpeg` and expects the painted frame back as an image. The server behind it is not included.
- **Frame rate.** The camera view moves at the screen's rate; the paint refreshes as fast as the phone can run the network, which on most phones will be a few frames a second rather than 24 to 30.
- **Tracking is 2D.** The painting follows pans and tilts. It does not model depth, so fast walking or close objects moving on their own will smear until the next painted frame.
- **Not yet run on a phone.** The network, motion tracking and blending were tested outside a browser. The camera, WebGL compositor and segmenter have not been run on a device.

## Third-party pieces

- Style-transfer weights: Magenta arbitrary-image-stylization, TensorFlow.js port by Reiichiro Nakano (Apache 2.0)
- TensorFlow.js, MediaPipe Tasks Vision (Apache 2.0), Three.js (MIT), qrcode-generator (MIT), loaded from jsDelivr
- DeepLab v3 segmentation model, loaded from Google's MediaPipe model storage
- Fraunces typeface, from Google Fonts (OFL)
