# satoshis.place
<img src="https://i.imgur.com/XUo6fAX.jpg" />

## Introduction
Welcome to satoshis.place, if you're a developer and you are interested in making apps, services or bots you'll find information here on how to interact with the API.

**Note: You won't find the source code for satoshis.place here, the project is not open source and there are no plans to do publish the code yet**

If you find any bugs or have any suggestions at all, feel free to open an issue and obviously any PRs to contribute towards this documentation are also welcome!

## Overview
Satoshi's place is a type of Lightning Network powered app (Lapp) that runs an open, permissionless and collaborative art board. Each pixel costs 1 satoshi to paint and can be painted over and over again. If you haven't seen it yet, [go check it out first](https://satoshis.place).

The aim of the project is to give everyone a fun, easy and cheap way to experience the power of the Bitcoin Lightning Network. But there's more to it, with the advent of microstransactions and programmatic money, we can actually build Lapps that interact with each other and even provide value added services. In the case of satoshis.place, some of the interesting things to build are:

### Bots
- Upload and draw pictures
- Protect a drawing
- Animate a drawing

### Tools
- Different paint brush sizes
- Spray paint
- Shapes (circles, rectangles, stars)

### Analytics
- Activity heatmaps
- General stats (i.e. txs/day, pixels/day)
- Timelapses

All of ideas above can be implemented today in some form or another with the existing API. They can be built as separate services / websites / apps which can even monetize themselves by creating their own lightning invoices with a surchage for the service they provide.

If you need to setup a lightning network node, take a look at the excellent [lnd](https://github.com/lightningnetwork/lnd) and [c-lightning](https://github.com/ElementsProject/lightning).

## API
The endpoint for the mainnet API lives at https://api.satoshis.place. There is also a testnet version https://testnet-api.satoshis.place - you can use it to experiment with testnet coins before going for the real thing. Both APIs linked to https://satoshis.place and https://testnet.satoshis.place respectively.

The whole API is built using websockets (socket.io@^1.7.4). Here's a javascript example on how to setup a connection and subscribing to the events we expect to receive:
```
import io from 'socket.io-client'

const socket = io(API_URI)

// Listen for errors
socket.on('error', ({ message }) => {
  // Requests are rate limited by IP Address at 10 requests per second.
  // You might get an error returned here.
  console.log(message)
})

// Wait for connection to open before setting up event listeners
socket.on('connect', a => {
  console.log('API Socket connection established with id', socket.id)
  // Subscribe to events
  socket.on('GET_LATEST_PIXELS_RESULT', handleGetLatestPixelsResult)
  socket.on('NEW_ORDER_RESULT', handleNewOrderResult)
  socket.on('ORDER_SETTLED', handleOrderSettled)
  socket.on('GET_SETTINGS_RESULT', handleGetSettingsResult)
})

// Here's two examples on how you send a request, the response will be
// in the callbacks above.
socket.emit('GET_LATEST_PIXELS')
socket.emit('NEW_ORDER', pixelsArray)


```

There are 3 events that you can send and 4 you can listen to, we'll go over them now. All send + receive events, except for ORDER_SETTLED, are only between a single client and the server. You socket session ID is what allows the server to know who to respond to.

### Send Events

#### `GET_LATEST_PIXELS`
Request an image uri for the latest state of the board. No data needs to be sent. The response will be received in `GET_LATEST_PIXELS_RESULT`.

#### `GET_SETTINGS`
Request settings like invoice expiry, allowed colors etc. No data needs to be sent. The response will be received in `GET_SETTINGS_RESULT`.

#### `NEW_ORDER`
When you want to draw something, send a request with this event and an array of objects like:
```
[
  {
    coordinates: [0, 0],
    color: '#ffffff'
  },
  ...
]
```
where `coordinates` is the x, y position in the board (min: 0, max: 1000 for both values), and `color` is one of the allowed colors received in the settings, in web hex format. Each object represents a pixel, there's a limit to the number of pixels you can submit in each order, this is determined by the `orderPixelsLimit` value in settings.

### Receive Events

All receive events have a payload in the shape of `{ data: ..., error: ... }`, if `error` is set it will be a string with a message about an error that occured. The stuff you'll care about will be in `data`.

#### `GET_LATEST_PIXELS_RESULT`
Received after making a `GET_LATEST_PIXELS` request. Contains a base64 png image uri that represents the canvas in its current state.

#### `GET_SETTINGS_RESULT`
Received after making a `GET_SETTINGS` request. Example `data` object:
```
{
  boardLength: 1000,
  colors: ['#ffffff', '#e4e4e4', ...],
  invoiceExpiry: 600,
  orderPixelsLimit: 250000,
  pricePerPixel: 1
}
```
_Note: There are 16 colors available to use._

#### `NEW_ORDER_RESULT`
Received after making a successful order. Contains the generated lightning payment request which you will pay to finalize your drawing. Example `data`:
```
{
	data: 'lnbc110n1pdjpn47pp5up...'
}
```
#### `ORDER_SETTLED`
This event is used to notify all users that an update has occured on the board. `data` will look like this:
```
{
  image: 'data:image/png;base64,iVBOR...',
  paymentRequest: 'lnbc110n1pdjpn47pp5up...',
  pixelsPaintedCount: 29,
  sessionId: "ck0ehHuJ0Y2fLEMBAARS" // The session id of the user who just paid, this is used in lieu of a username to display in the satoshis.place hud.
}
```
