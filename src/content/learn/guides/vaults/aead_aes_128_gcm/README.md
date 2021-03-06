---
order: 6
title: AEAD AES 128 GCM
---

# AEAD AES 128 GCM

The AEAD operation used in this example is `AEAD_AES_128_GCM`, which is
precisely defined in [RFC 5116](https://tools.ietf.org/html/rfc5116).

## AEAD AES 128 GCM Encrypt

Using an ockam secret containing the AES128 key we can use the encrypt
function to generate an encrypted ciphertext. The output of the encrypt function
is the encrypted ciphertext + a 16 byte tag attached to the end of the ciphertext.
This means when creating a buffer for the output of encrypt, the buffer must be
16 bytes longer than the plaintext that's being encrypted.

```c
uint16_t nonce = 1;

char*  additional_data        = "some metadata that will be authenticated but not encrypted";
size_t additional_data_length = strlen(additional_data);

char*  plaintext        = "some data that will be encrypted";
size_t plaintext_length = strlen(plaintext);

size_t   ciphertext_and_tag_size = plaintext_length + OCKAM_VAULT_AEAD_AES_GCM_TAG_LENGTH;
uint8_t* ciphertext_and_tag;
size_t   ciphertext_and_tag_length;

error = ockam_memory_alloc_zeroed(&memory, &ciphertext_and_tag, ciphertext_and_tag_size);
if (error) goto exit;

error = ockam_vault_aead_aes_gcm_encrypt(&vault,
                                         &key,
                                         nonce,
                                         (uint8_t*) additional_data,
                                         additional_data_length,
                                         (uint8_t*) plaintext,
                                         plaintext_length,
                                         ciphertext_and_tag,
                                         ciphertext_and_tag_size,
                                         &ciphertext_and_tag_length);
if (error) goto exit;
```

## AEAD AES 128 GCM Decrypt

To decrypt the ciphertext and tag data, the device receiving the data would need to have arrived
at the same AES128 key through a key agreement scheme first. Using the same AES128 key, along with
the same nonce value and additional data, the ciphertext and tag data can be passed into decrypt.
The result will be a buffer 16 bytes less than the ciphertext and tag, and will contain the decrypted
message.

```c
uint16_t nonce = 1;

char*  additional_data        = "some metadata that will be authenticated but not encrypted";
size_t additional_data_length = strlen(additional_data);

size_t   decrypted_plaintext_size = plaintext_length;
uint8_t* decrypted_plaintext;
size_t   decrypted_plaintext_length;

error = ockam_memory_alloc_zeroed(&memory, &decrypted_plaintext, decrypted_plaintext_size);
if (error) goto exit;

error = ockam_vault_aead_aes_gcm_decrypt(&vault,
                                         &key,
                                         nonce,
                                         (uint8_t*) additional_data,
                                         additional_data_length,
                                         ciphertext_and_tag,
                                         ciphertext_and_tag_length,
                                         decrypted_plaintext,
                                         decrypted_plaintext_size,
                                         &decrypted_plaintext_length);
if (error) goto exit;
```

## Complete Example

This example shows the following:
* Generate 16 bytes of random data to use for an AES key.
* Create an ockam vault secret using the 16 bytes of random data.
* Encrypt a payload using the AES key, additional data and nonce.
* Decrypt the ciphertext + tag using the AES key, additional data and nonce.
* Print out the resuling ciphertext + tag and the decrypted data.

```c

#include "ockam/error.h"

#include "ockam/memory.h"
#include "ockam/memory/stdlib.h"

#include "ockam/random.h"
#include "ockam/random/urandom.h"

#include "ockam/vault.h"
#include "ockam/vault/default.h"

#include <stdio.h>

int main(void)
{
  int exit_code = 0;

  ockam_error_t error        = OCKAM_ERROR_NONE;
  ockam_error_t deinit_error = OCKAM_ERROR_NONE;

  ockam_memory_t                   memory           = { 0 };
  ockam_random_t                   random           = { 0 };
  ockam_vault_t                    vault            = { 0 };
  ockam_vault_default_attributes_t vault_attributes = { .memory = &memory, .random = &random };

  error = ockam_memory_stdlib_init(&memory);
  if (error != OCKAM_ERROR_NONE) { goto exit; }

  error = ockam_random_urandom_init(&random);
  if (error != OCKAM_ERROR_NONE) { goto exit; }

  error = ockam_vault_default_init(&vault, &vault_attributes);
  if (error != OCKAM_ERROR_NONE) { goto exit; }

  /*
   * Generate a 16 byte random number to use as the AES key
   */

  uint8_t buffer[16];

  error = ockam_vault_random_bytes_generate(&vault, &buffer[0], 16);
  if(error != OCKAM_ERROR_NONE) goto exit;

  /*
   * Import the 16 byte random data into a vault secret
   */

  ockam_vault_secret_t            key            = { 0 };
  ockam_vault_secret_attributes_t key_attributes = { 0 };

  key_attributes.length      = 16;
  key_attributes.type        = OCKAM_VAULT_SECRET_TYPE_AES128_KEY;
  key_attributes.purpose     = OCKAM_VAULT_SECRET_PURPOSE_KEY_AGREEMENT;
  key_attributes.persistence = OCKAM_VAULT_SECRET_EPHEMERAL;

  error = ockam_vault_secret_import(&vault,
                                    &key,
                                    &key_attributes,
                                    &buffer[0],
                                    16);
  if(error != OCKAM_ERROR_NONE) goto exit;


  /* Encrypt the plaintext using the the key, additional data and nonce. */

  uint16_t nonce = 1;

  char*  additional_data        = "some metadata that will be authenticated but not encrypted";
  size_t additional_data_length = strlen(additional_data);

  char*  plaintext        = "some data that will be encrypted";
  size_t plaintext_length = strlen(plaintext);

  size_t   ciphertext_and_tag_size = plaintext_length + OCKAM_VAULT_AEAD_AES_GCM_TAG_LENGTH;
  uint8_t* ciphertext_and_tag;
  size_t   ciphertext_and_tag_length;

  error = ockam_memory_alloc_zeroed(&memory, &ciphertext_and_tag, ciphertext_and_tag_size);
  if (error) goto exit;

  error = ockam_vault_aead_aes_gcm_encrypt(&vault,
                                           &key,
                                           nonce,
                                           (uint8_t*) additional_data,
                                           additional_data_length,
                                           (uint8_t*) plaintext,
                                           plaintext_length,
                                           ciphertext_and_tag,
                                           ciphertext_and_tag_size,
                                           &ciphertext_and_tag_length);
  if (error) goto exit;

  printf("Encrypted ciphertext and tag : ");
  for(int i = 0; i < ciphertext_and_tag_size; i++) {
    printf("%02x", ciphertext_and_tag[i]);
  }
  printf("\r\n");

  /*
   * Decrypt the ciphertext + tag using the same key, additional data and nonce.
   */

  size_t   decrypted_plaintext_size = plaintext_length;
  uint8_t* decrypted_plaintext;
  size_t   decrypted_plaintext_length;

  error = ockam_memory_alloc_zeroed(&memory, &decrypted_plaintext, decrypted_plaintext_size);
  if (error) goto exit;

  error = ockam_vault_aead_aes_gcm_decrypt(&vault,
                                           &key,
                                           nonce,
                                           (uint8_t*) additional_data,
                                           additional_data_length,
                                           ciphertext_and_tag,
                                           ciphertext_and_tag_length,
                                           decrypted_plaintext,
                                           decrypted_plaintext_size,
                                           &decrypted_plaintext_length);
  if (error) goto exit;


  printf("Decrypted plaintext          : ");
  printf("%s\r\n", decrypted_plaintext);

exit:

  ockam_memory_free(&memory, ciphertext_and_tag, ciphertext_and_tag_size);
  ockam_memory_free(&memory, decrypted_plaintext, decrypted_plaintext_size);
  ockam_vault_secret_destroy(&vault, &key);

  deinit_error = ockam_vault_deinit(&vault);
  ockam_random_deinit(&random);
  ockam_memory_deinit(&memory);

  if (error == OCKAM_ERROR_NONE) { error = deinit_error; }
  if (error != OCKAM_ERROR_NONE) { exit_code = -1; }
  return exit_code;
}
```

