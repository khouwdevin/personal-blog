---
title: "Walkie Talkie"
desc: "How to create simple walkie talkie with Next js and Livekit."
publishedAt: "2024-08-27"
updatedAt: "2025-07-16"
---

This blog will guide you to build a walkie talkie web app with Next js, Chakra UI and Livekit.

I am using Livekit because it is easy and open source, which means you can use your own server to deploy the Livekit service. But, in this case I am using the Livekit cloud because I could not run the server successfully and I thought to deploy it to Vercel, so it is better to use the cloud. However, if you are going to use this on local network you can try without the Livekit cloud.

# Set up

To begin with, let's create next app first using this code in your terminal.

```bash
npx create-next-app@latest walkie-talkie
```

You can follow this setting:

```bash
Need to install the following packages:
create-next-app@14.2.6
Ok to proceed? (y) y

√ Would you like to use TypeScript? ... No / Yes(*)
√ Would you like to use ESLint? ... No(*) / Yes
√ Would you like to use Tailwind CSS? ... No(*) / Yes
√ Would you like to use `src/` directory? ... No(*) / Yes
√ Would you like to use App Router? (recommended) ... No / Yes(*)
√ Would you like to customize the default import alias (@/*)? ... No(*) / Yes
```

After that, you will wait for the Next js app to finish installing, then you can move to the walkie-talkie directory and install Chakra UI and Livekit.

```bash
npm i @chakra-ui/react @emotion/react @emotion/styled framer-motion react/icons
```

```bash
npm i @livekit/components-react @livekit/components-styles livekit-client livekit-server-sdk
```

With all packages are installed, we can start setting up Chakra UI in Next js app. Open your code editor and put this code in your root layout.tsx.

```jsx
// app/layout.tsx

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

```jsx
// libs/chakraprovider.tsx

'use client'

import { ChakraProvider } from '@chakra-ui/react'

export function Providers({ children }: { children: React.ReactNode }) {
  return <ChakraProvider>{children}</ChakraProvider>
}
```

Do not forget to sign up on [Livekit](https://cloud.livekit.io/) to get API key and API secret, then you need to save it to .env file and put it on the root folder.

# UI and Methods

After it is done, we can continue to implement UI, you can delete all app/page.tsx and change into this code.

```jsx
// app/page.tsx

<>
    <audio style={{ display: "none" }} ref={audioRef}/>

    <Center height="100vh">
        <Stack alignItems="center">
            <Box>
                <IconButton aria-label="mic" icon={mic ? <FaMicrophone/> : <FaMicrophoneSlash/>} 
                borderRadius="full" boxSize={["250px", "300px"]} fontSize={["80px", "100px"]} _hover={{ boxShadow: "none" }}
                onClick={enableMic} isDisabled={status.isError || status.isLoading}/>
            </Box>

            <Stack pt={4} direction="row" alignItems="center" gap={4}>
                { 
                status.isLoading ?
                    <Spinner size="lg"/>              
                :
                (
                    (status.isConnected) ? 
                    <Icon aria-label="connected" as={MdOutlineSignalWifiStatusbar4Bar} fontSize={["30px", "40px"]}/> 
                    : 
                    <Icon aria-label="not connected" as={MdOutlineSignalWifiStatusbarNull} fontSize={["30px", "40px"]}/>
                )
                }

                <Text fontWeight="bold" fontSize={[25, 35]}>{status.message}</Text>
            </Stack>
        </Stack>
    </Center>
</>
```

Okay, the UI has been implemented, next we will fill the missing part for the functions and the states, I will breakdown one by one. 

We will have a big icon button in the middle of the screen to control mic, it will mute and umute the mic. Below the icon button, we have text as information, it will be connected, connecting to room or error, all of that will contain in status state. The other states will be room, from Livekit room and we need to remember the mic state it will be boolean and audioRef for refrence from audio tag in component.

```jsx
// app/page.tsx

const [status, setStatus] = useState({
    isConnected: false,
    isLoading: true,
    isError: false,
    message: "connecting..."
})

const [room, setRoom] = useState<Room>()

const [mic, setMic] = useState<boolean>(false)    

const audioRef = useRef<HTMLAudioElement>(null)
```

Move on into methods, we have enableMic, its role to switch mic on and off, the other one is error which it will set the status state to error condition and will change the UI as information for the user.

```jsx
// app/page.tsx

const error = (message: string) => {    
    setStatus({
        isLoading: false,
        isError: true,
        isConnected: false,
        message: message
    })
}

const enableMic = async () => {
    if (!room) return

    const currentMicState = !mic

    await room.localParticipant.setMicrophoneEnabled(!mic)

    setMic(currentMicState)
}
```

The last thing in page.tsx we have useEffects for subscribing to RoomEvent and also connect to room when user visit the page. For the first useEffect, it will connect user to the Livekit cloud and create room which in this occasion we will have just one room but it is possible for you to create multiple rooms if you want. If you see other services, they usually mute the mic at first, but I found issue with ```localParticipant.setMicrophoneEnabled(false)```, it will not let you subscribe to other participants audio and makes you cannot hear any sound before you turn on your mic, so the solution is set it true first and then set it false.

```jsx
// app/page.tsx

useEffect(() => {
  const getRoom = async () => {
      try {
          const res = await fetch("/api/room", {
              method: "POST"
          })
          const { token } = await res.json()

          const currentRoom = new Room({
              audioCaptureDefaults: {
                  autoGainControl: true,
                  deviceId: "",
                  echoCancellation: true,
                  noiseSuppression: true,
              },
              publishDefaults: {
                  audioPreset: {
                      maxBitrate: 20_000
                  }
              }
          })

          setRoom(currentRoom)

          await currentRoom.connect(process.env.LIVEKIT_URL as string, token)

          await currentRoom.localParticipant.setMicrophoneEnabled(true)
          await currentRoom.localParticipant.setMicrophoneEnabled(false)
      } catch {
          error("room error")
      }
  }
  
  getRoom()
, []}
  
```

And for RoomEvent, it only set message to user and connect to audio through audioRef.

```jsx
// app/page.tsx

useEffect(() => {
    if (!room || !audioRef.current) return

    room.on(RoomEvent.Connected, () => {
        setStatus({ 
            isError: false, 
            isLoading: false, 
            isConnected: true, 
            message: "walkie talkie room"
        })

        setTimeout(() => {
            setStatus((prev) => ({ 
                ...prev, 
                message: "connected" 
            }))
        }, 1000)
    })

    room.on(RoomEvent.Disconnected, async (reason) => {
        setStatus({ 
            isError: true, 
            isLoading: false, 
            isConnected: false, 
            message: "disconnected"
        })
    })

    room.on(RoomEvent.Reconnecting, () => {
        setStatus({
            isError: false,
            isLoading: true,
            isConnected: false,
            message: "reconnecting"
        })
    })

    room.on(RoomEvent.Reconnected, () => {
        setStatus({ 
            isError: false, 
            isLoading: false,
            isConnected: true,
            message: "reconnected to room" 
        })

        setTimeout(() => {
            setStatus((prev) => ({ 
                ...prev,
                message: "connected" 
            }))
        }, 1000)
    })

    room.on(RoomEvent.TrackSubscribed, (track, publication, participant) => {
        if (!audioRef.current) return

        track.attach(audioRef.current)
    })

    room.on(RoomEvent.TrackUnsubscribed, (track, publication, participant) => {
        if (!audioRef.current) return

        track.detach(audioRef.current)
    })
}, [room])
```

It all finish, you can see the complete code below.

```jsx
// app/page.tsx

'use client'

import { Box, Center, Icon, IconButton, Spinner, Stack, Text } from "@chakra-ui/react";
import { Room, RoomEvent } from "livekit-client";
import { useEffect, useRef, useState } from "react";
import { FaMicrophone, FaMicrophoneSlash } from "react-icons/fa";
import { MdOutlineSignalWifiStatusbar4Bar, MdOutlineSignalWifiStatusbarNull } from "react-icons/md";

export default function Page() {
    const [status, setStatus] = useState({
        isConnected: false,
        isLoading: true,
        isError: false,
        message: "connecting..."
    })

    const [room, setRoom] = useState<Room>()

    const [mic, setMic] = useState<boolean>(false)    

    const audioRef = useRef<HTMLAudioElement>(null)

    const error = (message: string) => {    
        setStatus({
            isLoading: false,
            isError: true,
            isConnected: false,
            message: message
        })
    }

    const enableMic = async () => {
        if (!room) return

        const currentMicState = !mic

        await room.localParticipant.setMicrophoneEnabled(!mic)

        setMic(currentMicState)
    }

    const getRoom = async () => {
        try {
            const res = await fetch("/api/room", {
                method: "POST"
            })
            const { token } = await res.json()

            const currentRoom = new Room({
                audioCaptureDefaults: {
                    autoGainControl: true,
                    deviceId: "",
                    echoCancellation: true,
                    noiseSuppression: true,
                },
                publishDefaults: {
                    audioPreset: {
                        maxBitrate: 20_000
                    }
                }
            })

            setRoom(currentRoom)

            await currentRoom.connect(process.env.LIVEKIT_URL as string, token)

            await currentRoom.localParticipant.setMicrophoneEnabled(true)
            await currentRoom.localParticipant.setMicrophoneEnabled(false)
        } catch {
            error("room error")
        }
    }

    useEffect(() => {
        if (!room || !audioRef.current) return

        room.on(RoomEvent.Connected, () => {
            setStatus({ 
                isError: false, 
                isLoading: false, 
                isConnected: true, 
                message: "walkie talkie room"
            })

            setTimeout(() => {
                setStatus((prev) => ({ 
                    ...prev, 
                    message: "connected" 
                }))
            }, 1000)
        })

        room.on(RoomEvent.Disconnected, async (reason) => {
            setStatus({ 
                isError: true, 
                isLoading: false, 
                isConnected: false, 
                message: "disconnected"
            })
        })

        room.on(RoomEvent.Reconnecting, () => {
            setStatus({
                isError: false,
                isLoading: true,
                isConnected: false,
                message: "reconnecting"
            })
        })

        room.on(RoomEvent.Reconnected, () => {
            setStatus({ 
                isError: false, 
                isLoading: false,
                isConnected: true,
                message: "reconnected to room" 
            })

            setTimeout(() => {
                setStatus((prev) => ({ 
                    ...prev,
                    message: "connected" 
                }))
            }, 1000)
        })

        room.on(RoomEvent.TrackSubscribed, (track, publication, participant) => {
            if (!audioRef.current) return

            track.attach(audioRef.current)
        })

        room.on(RoomEvent.TrackUnsubscribed, (track, publication, participant) => {
            if (!audioRef.current) return

            track.detach(audioRef.current)
        })
    }, [room])

    useEffect(() => {
        getRoom()
    }, [])

    return (
        <>
            <audio style={{ display: "none" }} ref={audioRef}/>

            <Center height="100vh">
                <Stack alignItems="center">
                    <Box>
                        <IconButton aria-label="mic" icon={mic ? <FaMicrophone/> : <FaMicrophoneSlash/>} 
                        borderRadius="full" boxSize={["250px", "300px"]} fontSize={["80px", "100px"]} _hover={{ boxShadow: "none" }}
                        onClick={enableMic} isDisabled={status.isError || status.isLoading}/>
                    </Box>

                    <Stack pt={4} direction="row" alignItems="center" gap={4}>
                        { 
                        status.isLoading ?
                            <Spinner size="lg"/>              
                        :
                        (
                            (status.isConnected) ? 
                            <Icon aria-label="connected" as={MdOutlineSignalWifiStatusbar4Bar} fontSize={["30px", "40px"]}/> 
                            : 
                            <Icon aria-label="not connected" as={MdOutlineSignalWifiStatusbarNull} fontSize={["30px", "40px"]}/>
                        )
                        }

                        <Text fontWeight="bold" fontSize={[25, 35]}>{status.message}</Text>
                    </Stack>
                </Stack>
            </Center>
        </>
    )
}

```

# API

From the client side, we can do the next step to server side where we need to create API to get token for connecting to a room. All you need to do is creating folder ```/app/api``` and ```/app/api/room```, also a file called route.ts in ```/app/api/room```. 

```ts
// app/api/room/route.ts

import { AccessToken } from "livekit-server-sdk";
import { NextResponse } from "next/server";
import crypto from "crypto"

export async function POST() {
    try {
      const username = crypto.randomBytes(8).toString("hex")
      const roomName = "walkie-talkie"

      const at = new AccessToken(
        process.env.LIVEKIT_API_KEY,
        process.env.LIVEKIT_API_SECRET,
        {
          identity: username,
        },
      )
      at.addGrant({ roomJoin: true, room: roomName })

      const token = await at.toJwt()

      return NextResponse.json({ token })
    } catch(e) {
      return NextResponse.json({ error: e })
    }
}
```

I am using crypto library to create a random username, but if you want to use an account for each user you can do that by inserting the user uuid or the other username to the username. Also, if you have more than one room you can pass the room name to the api.

# Result

It all set, you can start deploy the app to Vercel or other services and then start using the app. 

![Walkie Talkie Page](https://cdn.khouwdevin.com/blog/images/e3c99b50-38bb-4958-9901-d92e3bff8248)

The code will be available here on my Github [Walkie-Talkie](https://github.com/khouwdevin/walkie-talkie). I have two folders, the second one is for backend, it is basically the same as the api route, but I add more features like websocket to send messages, but it is not done yet, if you want me to finish the backend you can comment bellow and I will make a new blog about it.

Happy coding!