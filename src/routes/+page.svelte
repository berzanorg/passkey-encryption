<script lang="ts">
  let passkeyId = $state<ArrayBuffer>()

  let data = $state('very private data')

  let encryptedData = $state<string>()

  let decryptedData = $state<string>()

  const register = async () => {
    const rCredential = (await navigator.credentials.create({
      publicKey: {
        challenge: new Uint8Array([1, 2, 3, 4]),
        rp: {
          name: 'Passkey Encryption',
          id: 'passkey-encryption.pages.dev',
        },
        user: {
          id: new Uint8Array([5, 6, 7, 8]),
          name: 'Passkey Encryption',
          displayName: 'Passkey Encryption',
        },
        pubKeyCredParams: [
          { alg: -8, type: 'public-key' }, // Ed25519
          { alg: -7, type: 'public-key' }, // ES256
          { alg: -257, type: 'public-key' }, // RS256
        ],
        authenticatorSelection: {
          userVerification: 'required',
        },
        extensions: {
          prf: {
            eval: {
              first: new Uint8Array(new Array(32).fill(1)).buffer,
            },
          },
        },
      },
    })) as PublicKeyCredential | null

    if (rCredential === null) return

    if (rCredential.getClientExtensionResults().prf?.enabled !== true) {
      alert('PRF extension is not supported on your browser.')
      return
    }

    passkeyId = rCredential.rawId
  }

  const encrypt = async () => {
    const gCredential = (await navigator.credentials.get({
      publicKey: {
        challenge: new Uint8Array([9, 0, 1, 2]),
        allowCredentials: [
          {
            id: passkeyId!,
            transports: ['internal'],
            type: 'public-key',
          },
        ],
        rpId: 'passkey-encryption.pages.dev',
        userVerification: 'required',
        extensions: {
          prf: {
            eval: {
              first: new Uint8Array(new Array(32).fill(1)).buffer,
            },
          },
        },
      },
    })) as PublicKeyCredential | null

    if (gCredential === null) return

    const key = gCredential.getClientExtensionResults().prf?.results?.first

    if (!key) return

    const keyMaterial = new Uint8Array(key as ArrayBuffer)

    const keyDerivationKey = await crypto.subtle.importKey('raw', keyMaterial, 'HKDF', false, ['deriveKey'])

    const info = new TextEncoder().encode('encryption key')

    const salt = new Uint8Array()

    const encryptionKey = await crypto.subtle.deriveKey(
      { name: 'HKDF', info, salt, hash: 'SHA-256' },
      keyDerivationKey,
      { name: 'AES-GCM', length: 256 },

      false,
      ['encrypt', 'decrypt']
    )

    console.log(encryptionKey)

    const encrypted = await crypto.subtle.encrypt(
      { name: 'AES-GCM', iv: new Uint8Array(12).fill(1) },
      encryptionKey,
      new TextEncoder().encode(data)
    )

    encryptedData = Array.from(new Uint8Array(encrypted))
      .map((byte) => byte.toString(16).padStart(2, '0'))
      .join('')
  }

  const decrypt = async () => {
    const encrypted = new Uint8Array(encryptedData!.length / 2)
    for (let i = 0; i < encryptedData!.length; i += 2) {
      encrypted[i / 2] = parseInt(encryptedData!.slice(i, i + 2), 16)
    }

    const gCredential = (await navigator.credentials.get({
      publicKey: {
        challenge: new Uint8Array([9, 0, 1, 2]),
        allowCredentials: [
          {
            id: passkeyId!,
            transports: ['internal'],
            type: 'public-key',
          },
        ],
        rpId: 'passkey-encryption.pages.dev',
        userVerification: 'required',
        extensions: {
          prf: {
            eval: {
              first: new Uint8Array(new Array(32).fill(1)).buffer,
            },
          },
        },
      },
    })) as PublicKeyCredential | null

    if (gCredential === null) return

    const key = gCredential.getClientExtensionResults().prf?.results?.first

    if (!key) return

    const keyMaterial = new Uint8Array(key as ArrayBuffer)

    const keyDerivationKey = await crypto.subtle.importKey('raw', keyMaterial, 'HKDF', false, ['deriveKey'])

    const info = new TextEncoder().encode('encryption key')

    const salt = new Uint8Array()

    const encryptionKey = await crypto.subtle.deriveKey(
      { name: 'HKDF', info, salt, hash: 'SHA-256' },
      keyDerivationKey,
      { name: 'AES-GCM', length: 256 },

      false,
      ['encrypt', 'decrypt']
    )

    const decrypted = await crypto.subtle.decrypt(
      { name: 'AES-GCM', iv: new Uint8Array(12).fill(1) },
      encryptionKey,
      encrypted
    )

    decryptedData = new TextDecoder().decode(decrypted)
  }
</script>

<header
  class="sticky top-0 h-16 z-30 flex items-center justify-between w-full px-4 bg-[#222] border-b border-[#333333]"
>
  <a href="/" class="text-2xl font-bold flex items-center gap-2"
    ><img src="/person-key.svg" class="h-6" alt="logo" />Passkey Encryption</a
  >

  <a href="https://github.com/berzanorg/passkey-encryption" target="_blank"
    ><img src="/github.svg" class="h-5" alt="github" /></a
  >
</header>

<main class="flex flex-col w-full px-4 max-w-xl gap-10 flex-1">
  <div class="flex flex-col gap-4">
    <p class="font-semibold text-xl">Register Passkey</p>
    <button disabled={!!passkeyId} onclick={register}>
      {#if !!passkeyId}
        Registered With Face ID <img src="/face-id.svg" alt="face id" class="w-6" />
      {:else}
        Register With Face ID <img src="/face-id.svg" alt="face id" class="w-6" />
      {/if}
    </button>
  </div>
  <div class="flex flex-col gap-4">
    <div class="flex flex-col gap-2">
      <p class="font-semibold text-xl">Data To Encrypt</p>
      <input
        bind:value={data}
        oninput={() => {
          encryptedData = undefined
          decryptedData = undefined
        }}
      />
    </div>
    <button disabled={!passkeyId} onclick={encrypt}
      >Encrypt With Face ID <img src="/lock.svg" alt="face id" class="w-6" /></button
    >
  </div>
  <div class="flex flex-col gap-4">
    <div class="flex flex-col gap-2">
      <p class="font-semibold text-xl">Data To Decrypt</p>
      <input placeholder="Decrypted form will appear here..." disabled value={encryptedData} />
    </div>
    <button disabled={!encryptedData} onclick={decrypt}
      >Decrypt With Face ID <img src="/unlock.svg" alt="face id" class="w-6" /></button
    >
  </div>
  <div class="flex flex-col gap-2">
    <p class="font-semibold text-xl">Result</p>
    <input placeholder="Result will appear here..." disabled value={decryptedData} />
  </div>
</main>

<footer class="w-full px-4 py-8 max-w-xl">
  <p class="text-[#bbbbbb]">
    Built and designed by <a href="https://x.com/berzanorg" class="text-indigo-400">@berzanorg</a> in Korea.
  </p>
</footer>
