# `vaultfile` - A basic shared secret manager

Many times, programs need to contain within their source a set of shared secrets.
More often then not, they're stored in plain-text, just because nothing out there is convenient enough to use.

Tools like Ansible simply use a symmetric key that must be shared with all the people involved, making hard to have finer-grained control on access and makes it hard to revoke credentials.

Other tools with more rich control of secrets require a server, which is additional overhead and may be too complex/expensive for a smaller team.

With `vaultfile` I wanted to take a simpler approach. The core ideas are listed below:
1. **Secret storage should be contained within a vaultfile.** No servers should be needed! It should be possible to safely commit this file to source control.
2. **Access to the secret should be granted on a person-by-person basis**. Asymmetric cryptography should be used to allow multiple people to have individual access to the shared secret(s). The public keys of all allowed parties should also be included inside the secrets file, to make it easy to add new shared secrets to the file without needing to consciously store everybody's keys.
3. **Revocation of a secret should be manageable in some way**. Under the assumptions of the two previous points:
    * Everyone who has the vaultfile & a private key corresponding to one of the public keys registered inside the vaultfile has perpetual access to the all the secrets in the vaultfile, since the person could have copied the secret to another place (this is the case with any shared secret system).
    * the only way for the secret to be truly "revoked" is to change the shared secret. This is something that, in my opinion, should always be the case when revoking a shared secret under any scheme. The proposed revocation process will be shared in more detail in the corresponding implementation section.

## Implementation

The vaultfile is a serialized JSON file with the following sections:
- A list of public keys that are granted access to the shared secrets.
    - These keys are simple JSON serializations of the Rust `RSAPublicKey` struct, so they are just JSON objects as well (not PEM, etc.)
- A list of shared secrets. Each shared secret contains:
    - the **secret**, encrypted with a symmetric key, serialized as a Base64 string.
    - the **encrypted symmetric key**, itself encrypted with all of the public keys registered in the vaultfile. **Any secret should have an entry here for _each and every_ public key registered in the vaultfile.**

In the examples it is shown with a `.vault` extension, but this is merely a suggestion. Any file extension will be accepted by the tool without complaints as long as it is a readable vaultfile.

### Private/public key storage

Vaultfile private/public keys can be generated/stored anywhere. However, by default, they will be stored in the users `$HOME` directory, under `$HOME/.vaultfile/`. For Windows environments where `$HOME` is not defined, Vaultfile will fallback onto the `%USERPROFILE%` environment variable.

The default private key filename is `$USER.key` (along with the public key `$USER.key.pub`). On Windows environments where `$USER` is not defined, Vaultfile will fallback onto `%USERNAME%`.

### Workflow
First, create your private/public keypair, with vaultfile itself (more options in the Usage section):

    vaultfile generate-key

This command will create a key under `$HOME/.vaultfile/$USER.key` (and `$USER.key.pub` at the same location) (or `%USERPROFILE%/.vaultfile/%USERNAME%.key` on Windows)

To create a new vaultfile, use the register-key, and register the key you have just created:

    vaultfile register-key --file my_vaultfile.vault --key-name=$USER --key-file=$HOME/.vaultfile/$USER.key.pub

 Even if you specify the private key in this step (instead of the public key), only the public part will be saved in the vaultfile, so it's not a big deal to specify either one. The main reason of having the public file as well is for convenience.

Other/new keys can be registered without possessing a private key as long as the file contains no secrets. Once a secret has been added to the vaultfile, the person must have a private key corresponding to one of the public keys, since part of the process will be to re-encrypt all of the secrets of the file with the newly added public key.

### Adding a secret (password, API key, etc.)
When you're ready to add a secret value to the vaultfile, execute:

    vaultfile add-secret --file my_vaultfile.vault --name aws_key --value ABCDEFG1234567890

You can also pass in a Base64-encoded value with --base64-value, which will be decoded before storing the value.

### Reading a secret
You can read a secret from the vaultfile if you possess a private key which corresponds to one of the public keys registered in the vaultfile. If you have your private key stored in the default location, then it's simply a matter of executing:

    vaultfile read-secret -f my_vaultfile.vault -n aws_key

Note that for this example, the short forms of the arguments `--name` and `--file` were used. To see which arguments have short forms and what they are, you can always invoke `--help`.

The secret will be printed out to `stdout`. This allows easy integration with other tools. It shouldn't be hard to develop a library for other languages which can read/write vaultfiles as well.

### Removal of a key (revocation of access)
In its simplest form, it would consist of removal of the key from the list of registered keys, as well as all of the encrypted values for that key. To perform this action, execute:

    vaultfile deregister-key --file my_vaultfile.vault --key-name $user

However, as stated earlier, this could still mean that the person has a copy of the previously encrypted secret. The best way to fully "revoke" the secret is to change the secret value after de-registering the key, which will share this new value only with the registered keys.

Ideally, the users would always be reading the shared secret from the vaultfile directly, so the change, once the effective value is changed, should be transparent for everyone.

Note that you cannot de-register the last key from the vaultfile (it must have at least one key registered).

## Usage

To see a help message, you can invoke:

    ./vaultfile --help

for the basic binary, as well as for any of the subcommands:

    ./vaultfile generate-key --help
    ./vaultfile read-secret --help

In any case, the usage cases are detailed below:

### Generate a new keypair
To generate a new private/public keypair (necessary when running `vaultfile` for the first time), the `generate-key` subcommand should be issued. The following usages are possible:

    vaultfile generate-key
    vaultfile generate-key --key-name=key_name
    vaultfile generate-key --key-path=path/to/private_key_file

In the first case, it will generate the private & public keys in the default location.

The `--key-name` option is useful for generating a key in the default location, but with a name other than your username.

Finally, the `--key-path` option is for when you just want to generate the private & public keys in a location other than the default.

If the private or public key file exists, it will ask you if you want to overwrite it. To prevent the prompt from appearing, you can simply add `-y`/`--yes` or `-n`/`--no` to answer the question by default.

**NOTE:** The private key file should probably be set to be accessible only by you (`chmod 600`), or even better, set to read-only (with `chmod 400`) to prevent accidental damage/removal of the key (if you really want to remove it you can always `chmod` it back). However, since this is meant as a cross-platform tool, no UNIX permission scheme can be assumed, which leaves the protection of your private key entirely up to you.

### User/key management
#### Vaultfile creation & key/user registration
The `register-key` subcommand can register a new public key in the vaultfile (if the vaultfile does not exist, it will be created for you). There are two ways to specify the public key to add:

    vaultfile register-key -f secret_file.vault --key-name=<KEY_NAME> --key-file=<PATH_TO_PUBLIC_KEY_FILE>
    vaultfile register-key -f secret_file.vault --key-name=<KEY_NAME> --key-json=<PUBLIC_KEY_JSON_STRING>

The `<KEY_NAME>` string should be name you wish the key to have in the vaultfile. Normally it should be something like a username, or something else unique to you (so that other uses of the vaultfile will know who the key belongs to).

A public key can be added as a file (first option) or as a JSON string (second option), which could be obtained from another vaultfile (i.e. using the `show-key` subcommand to extract the JSON string for a specific public key) or easily copied from a chat log.

If a key with that name already exists in the vaultfile, a confirmation warning will appearing asking if it's OK to overwrite the key (can be overwritten without confirmation if the `-y` flag is added).

#### Registration of a new key in a vaultfile with secrets
If the vaultfile already contains secrets, there is one more element needed to register a new key: a valid private key whose public key is already registered in the vaultfile.

The reason is simple: any newly added key needs to gain access to all the secrets contained in the vaultfile (by design). Therefore, to register a new key, you need to be able to read all of the vaultfile's secrets, to re-encrypt them with the newly registered public key, thereby granting access to the new trusted party.

The parameter to specify this is `--private-key-name`, which should be the name of a file under the `~/.vaultfile` directory (or `%USERPROFILE%\.vaultfile` on Windows), with a `.key` extension (i.e. `--private-key-name mithrandir` will open the file `~/.vaultfile/mithrandir.key`). Usage template:

    vaultfile register-key -f secret_file.vault --key-name=<KEY_NAME> --key-file=<PATH_TO_PUBLIC_KEY_FILE> --private--key-name=<PRIVATE_KEY_NAME>
    vaultfile register-key -f secret_file.vault --key-name=<KEY_NAME> --key-json=<PUBLIC_KEY_JSON_STRING> --private-key-name=<PRIVATE_KEY_NAME>

#### Listing of the public keys registered in the vaultfile

List the keys registered in the vaultfile:

    vaultfile list-keys --file secret_file.vault

Will output a list of names of the keys registered in the vaultfile.

#### Show a public key registered in a vaultfile
To show the JSON-encoding of a public key registered in a vaultfile:

    vaultfile list-keys --file secret_file.vault --key-name gandalf

This will output a JSON string in one line to `stdout`, containing the numerical parameters of the public key registered in the vaultfile. Specifying a name that is not registered will output an error message on `stderr`.

#### De-register a public key from a vaultfile

De-register a public key from the vaultfile:

    vaultfile deregister-key -f secret_file.vault --key-name <KEY_NAME>

### Secret management
#### Write a secret to a vaultfile
To write/add a secret to a vaultfile, the following commands can be used:

    vaultfile add-secret --file secret_file.vault --name=<SECRET_NAME> --value=<SECRET_VALUE>
    vaultfile add-secret --file secret_file.vault --name=<SECRET_NAME> --base64-value=<BASE64_ENCODED_SECRET_VALUE>

if a secret with that name already exists in the vaultfile, a confirmation warning will appearing asking if it's OK to overwrite the secret (can be overwritten without confirmation if the `-y` flag is added).

The second option exists to input information that is difficult to encode/escape, giving you the option to easily ensure that what is saved is exactly what was intended.

Notice that a secret can be added to the vaultfile without being in possession of any of the access keys. This is because the secret is encrypted with the public keys registered in the file and will afterwards only be readable by someone who possesses the private key that corresponds to that public key.

#### Read a secret from the vaultfile
To read a secret from the vaultfile, the following command can be used:

    vaultfile read-secret --file secret_file.vault --name <SECRET_NAME> [--key-name <KEY_NAME>> | --key-file <KEY_FILE>] [--no-eol]

The result will be printed out to `stdout`.

You must be in possession of a private key corresponding to **one** of the public keys registered in the file, to be able to read the secret value.

If `--key-name` is provided, the private key will be assumed to be at `$HOME/.vaultfile/PROVIDED_KEY_NAME.key`. If `--key-file` is provided, the value will be assumed to be a relative (or absolute) path to the private key file. If neither is provided, then the key will be assumed to be at `$HOME/.vaultfile/$USER.key`. In all cases, `$HOME` will fallback to `%USERPROFILE%` and `$USER` to `%USERNAME%` on Windows environments (Cygwin environments, which have these variables defined, will work normally).

If you do not wish an end-of-line character to be printed after the secret value, add the flag `--no-eol` to the command invocation.

#### Delete a secret from the vaultfile

To delete a secret from the vaultfile, the following command can be used:

    vaultfile delete-secret --file secret_file.vault --name <SECRET_NAME>

This will remove the secret named SECRET_NAME from the vaultfile. No keys will be asked for. Since it's just a JSON file modification, this operation could also easily be performed manually, so it makes no sense to add artificial restrictions to this subcommand.

## Return codes
The standard BSD preferred exit codes were followed. [More information can be found here.](https://www.freebsd.org/cgi/man.cgi?query=sysexits&apropos=0&sektion=0&manpath=FreeBSD+11.2-stable&arch=default&format=html)

## Some design decisions

### No per-user/per-key secret access control
At first, no per-user secret access will be implemented. The main reason is that it would add more complexity (in my opinion) needlessly. If you want a different set of users (more or less restrictive) to have access to some other secrets, you can simply create a new vaultfile (it's free!)

If someone proposes a use-case where this sort of behavior is needed and the solution proposed in the previous paragraph is not good enough, we can think about implementing it then. For now, I think the existing features will do.
