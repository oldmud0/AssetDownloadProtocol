# Attorney Online Asset Download Protocol

## Why make it automatic?

Manual asset downloading is problematic in multiple ways:

 1. Changes to certain assets require players to download the entire package again.
 2. Custom assets between servers may conflict with one another, especially in case of custom emotes.
 3. Players don't know how to do it correctly.
 4. It becomes difficult to return to vanilla without re-downloading AO.

It is far more convenient, then, to programatically download the required assets for a server than 
to assume that all players have downloaded all files by hand in the correct manner.

This is, of course, an optional protocol that preserves backwards compatibility with other clients.
Clients that receive an asset download handshake request are not required to respond to it if they 
simply wish to to finish the main loading process.

## Benefits

 - Servers can propagate new character changes to players seamlessly.
 - Servers can alter the theme without modifying the theme for other servers.
 - Servers can restrict the unauthorized modification of characters.

## Bandwidth considerations

Naturally, finding a generic download service is much easier than finding a service that will allow
direct downloads without requiring some ludicrous payment (the only exception being Dropbox).
Depending on the circumstances, a server may either direct clients to download from a given link
(which must be a certain format, to prevent the propagation of malware), or the server will simply
send it directly to the client.

Either way, it is very possible that a client may repeatedly and maliciously download assets in an 
attempt to initiate a denial-of-service (DoS) attack; if this is a concern, it is recommended that
server software be designed to ban IP addresses temporarily for repeated failed handshakes, or to 
simply remove the opportunity to download assets from abusive clients by omitting the asset download
handshake request.

Servers can also reduce strain by moving away from GIF and WAV to other formats, such as FLIF
(not supported in AO2 yet), Ogg Vorbis, and more aggressively compressed GIFs.

## Filesystem

To provide the encapsulation that was previously mentioned, we need to change the way assets are
organized on the filesystem level. We can no longer place our images inside a big melting pot
of assets, as some servers might decide to modify the same character with the same name, or the
server might suddenly change the name of the character. We want to track the data, not the metadata,
in order to avoid deduplication.

Assume that the AO root folder is `/`. The conventional location for assets is inside `/base`, but
all assets downloaded through this protocol shall reside in `/download`, which will have:

 - a directory called `characters` storing characters, with each subdirectory named by the
   MD5 checksum of their contents (truncation to 24 characters is recommended regardless of hash);
     * A comment may be added to the name of the directory by following the hash with a hyphen and
	   the specified comment.
	 * Two directories with the same hash but different comments will cause undefined behavior.
 - another directory called `backgrounds` storing backgrounds in the same way as `characters`; and
 - a directory called `servers`, with each subdirectory named by the UUID of the server.
   (If the server has no UUID, one shall be generated using the IP address and port of the server.)
     * Server directories may store any asset that is not a character. (Characters inside these
	   directories will be ignored.)

### Server index

Each server will generate a file containing hashes of each of its character and background
directories, as well as a hash of each custom file (which will be placed in the server's directory
on the client side), excluding the server cache file itself.

```
[backgrounds]
58b643ff805be3ef530babbc birthday
4b2a14b27e4d5f83ef62ffdb default
1fd59b83635b192f21ace91d gs4

[characters]
fbae6980ee983f831b305b1c Adachi
d44e5c1d6da98a170d73432b Armstrong
...

[evidence]
9483c6ef2c8229e834dca4a4 1.gif
c1fe7b62e49019ef87fe117d 2.png
6665e848f2e02de1bc54e614 photo/agun.png
...

[misc]
...

[sounds]
...
```

The name of the directory must be preserved, as the data may already exist in `base`.
(To confirm, simply hash the data in the folder which the name points to in `base`.)

### Sample client-side file structure

```
/
	base/
		background/
		characters/
		evidence/
		misc/
		sounds/
	download/
		backgrounds/
			4c20ffaac4bacc08e6dc5781/
				...
		characters/
			3e25960a79dbc69b674cd4ec/
			64ec88ca00b268e5ba1a3567-Future-Luke/
				emotions/
				char.ini
				...
		servers/
			de9cd5dd-5adf-47d5-9856-ca18578ed818-Localhost/
			60462820-d773-40b1-8d29-9164898816c9/
				evidence/
				misc/
				sounds/
				server.txt
				index.txt
```

Note that in `base`, `background` is singular, whereas it is plural in the `download` directory.
Clearly, plural form is standard for everything else, so we shall frown at Fanat's general direction
for making `background` singular.

## Network protocol

On the `FL` packet, `assetdownload` must be included to begin the asset download process.

For conforming clients, immediately before the general loading process begins (e.g. `askchaa`), the
client must begin the asset download handshake by sending `AD#%` to the server. **This request must
not be sent or received at any other time.**

The server may respond in three possible ways:

 - For a direct data transfer:
   `AD2#<server uuid>#direct#<one-time code>#%`
 - For a data transfer on another server or port:
   `AD2#<server uuid>#remote#<ip>#<port>#<one-time code>#%`
 - No data transfers allowed; HTTP only:
   `AD2#<server uuid>#bigonly#%`

The one-time code is intended to prevent abuse of the protocol.

### Conflict resolution

The client will take this moment to determine whether a server directory exists that uses the
UUID generated by the IP address (i.e. orphan directory) as opposed to the UUID provided during the
handshake.

 - If the generated UUID exists as a directory, but not the real UUID, rename the directory to be
   the real UUID. (Be sure to preserve comments on the directory name.)
 - If only the real UUID exists as a directory, nothing needs to be done.
 - If both directories exist, delete the one with the generated UUID and leave the other one.
 - If neither directory exists, create a directory with the real UUID.

### File index transfer

The server will provide the file index.
(See the "[Server index](#server-index)" section for what the index looks like.)

`ADI#<raw file data>#%`

The client then writes the file index to `index.txt` in the server's directory.

### Big archives

The server may also send a string (i.e. a hash, or date, or number) representing the last time a
major update occurred.

It is up to the server owner to update this datum to something different to signify that clients
should download the largest archive possible instead of requesting individual assets.

For instance, the server owner should change the big archive string when new music is added or 
many characters are added (each character is around 600 KB to 2.5 MB in size). The server owner
should not change the string when most changes are deletions and the additions are very small.

After downloading the big archive, the client must still continue the download process to ensure
that all files are up-to-date.

**If `bigonly` has been chosen**, these conditions change: in this case, any discrepancy in the file
index will cause a download from the big archive. If assets are still missing after the download,
the client will ignore these errors and end the asset download process here.

`ADBIG#<server uuid>#<big archive string>#http#<URL>#%`

The server should wait a little while for the client to download and unpack the file.

### Determine what files must be downloaded

The client will look through folders and check which resources already exist.

It will first search by directory name in the corresponding `download/[resource type]` directory,
comparing them against hashes found in the file index. If the directory named with the hash is found
and the recomputed hash matches the hash in the index, then the asset exists.

If they are not found in `download`, then hashes will be taken in `base`, using the name column of
the index. If the name and hash match in `base`, then the asset exists.

The assets that could not be found, but are present in the index, are marked for download.
 
### Direct and remote data transfer

#### Handshake
This data transfer process begins with `ADTB#<password>#%`. A rejected password is an immediate
disconnect; otherwise, the server responds likewise with `ADTB#%`.

#### Transfer
Each TCP packet will only contain one header and one chunk of the file compressed using zlib.

An asset will be requested as `ADTF#<asset type>#<asset hash>#%`, and the response for each file
will have a header of `ADTF#<asset type>#<asset hash>#<filename>%`.

The rest of the packet will be a small chunk of the file, preferably 1024 bytes (or a consistent
size below the normal MTU of 1500 bytes). Chunks smaller than the fixed size must not pad the
remaining bytes with zeroes.

Subsequent packets will be preambled with `ADTFC#%`.

The file ends with `ADTFE#%`. The client makes no further requests until the asset ends with
`ADTAE#%`. The client may then make another request for an asset.

In a remote data transfer session, the client may disconnect from the server when there are no more
assets to download. In a local transfer session, the client simply continues to the next step in the
process.

### End
When the client is done with the data transfer and has effectively finished the asset download
process, the client simply sends `ADDONE#%`, which concludes the asset download process.
`server.txt` is written to disk.

## Local asset management

### server.txt

This file is generated in each server's directory to denote known server data. This includes:

 - Server name (`name`)
 - IP address (`ip`)
 - Last big archive string (`lastbig`)
 - Last connection time (`lastconnect`)

### index.txt

This file is generated in each server's directory as a local copy of the [server's index](#server-index).

### Manual download

Despite the automated nature of this protocol, sometimes server owners wish to push out a ZIP file
to be downloaded manually by players, perhaps placed at the end of a list of rules. In this case,
clients should have an option to select a ZIP file, enter the IP address of the server,
and unarchive it automatically, hashing the contents and placing them in their respective places.

Custom content that belongs in a server directory that does not exist yet should be placed in an
orphan directory, unless the root of the archive contains a `uuid.txt` file containing nothing
except the UUID of the server.

#### Orphan directories

These directories are named by a UUID generated from the IP address. Bearing no server.txt or
index.txt, they are intended to be picked up on connection to the correct server.

### Cleanup

To purge files from the download directory, the client simply iterates through each server and
deletes server directories that have lapsed a certain time since last connection.

The client iterates through the remaining servers again and looks for assets that are in use, based
on index.txt. The client then deletes assets that are not in use.

## Server owner responsibilities

Most things are already taken care of automatically by the server software, including the generation
of the index file. However, the owner must still ensure that the big archive link is up-to-date, and
that, when updated, the owner responsibly updates the big archive string so that clients get the
changes. The place of configuration is left to the server software.
