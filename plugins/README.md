# Email OAuth 2.0 Proxy plugins
Plugins are a semi-experimental Email OAuth 2.0 Proxy feature that enables the use of separate scripts to modify IMAP/POP/SMTP commands when they are received from the client or server before passing through to the other side of the connection.
This allows a wide range of additional capabilities or triggers to be added the proxy, as demonstrated in the examples below.


## Sample plugins
- [`Echo`](plugins/Echo.py): An example plugin that does nothing except call `log_debug` with all received messages.
As a result, it is not specific to IMAP, POP or SMTP, and can be used with any type of server.

- [`IMAPCapabilityModifier`](plugins/IMAPCapabilityModifier.py): An example plugin that intercepts a single IMAP `CAPABILITY` response and modifies it to remove support for `LITERAL+` (chosen only as a common example for demonstration purposes; this is not a useful plugin).

- [`IMAPCleanO365ATPLinks`](plugins/IMAPCleanO365ATPLinks.py): An example IMAP plugin that looks for Office 365 Advanced Threat Protection links (also known as Safe Links in Microsoft Defender for Office 365) and replaces them with their original values (i.e., removing the redirect).
As with most of the proxy's plugins, it would be more efficient to handle this on the server side (i.e., by disabling link modification), but this is not always possible.

- [`IMAPDisableDeflateCompression`](plugins/IMAPDisableDeflateCompression.py): An example IMAP plugin that looks for client requests to enable compression (RFC 1951 and 4978) and responds with NO every time, so that other plugins can continue to intercept requests.
Place this plugin before any others if you use a client that automatically tries to enable compression when it finds `COMPRESS=DEFLATE` in a `CAPABILITY` response.
An alternative option here if you do not need to actually edit messages is to keep compression enabled, but decompress within the plugin – see [`IMAPDecodeDeflateCompression`](plugins/IMAPDecodeDeflateCompression.py).

- [`IMAPDecodeDeflateCompression`](plugins/IMAPDecodeDeflateCompression.py): An example IMAP plugin that looks for client requests to enable compression (RFC 1951 and 4978) and, unlike [`IMAPDisableDeflateCompression`](plugins/IMAPDisableDeflateCompression.py), permits compression but decompresses messages within the plugin.
This allows monitoring and editing of compressed incoming and outgoing communication, but only within this plugin, not any others (and as a result only other deflate-aware plugins can be chained).
An alternative option and further improvement can be found in [`IMAPDeflateCompressionRemoteEnabler`](plugins/IMAPDeflateCompressionRemoteEnabler.py).

- [`IMAPDeflateCompressionRemoteEnabler`](plugins/IMAPDeflateCompressionRemoteEnabler.py): An example Email OAuth 2.0 Proxy IMAP plugin that looks for `COMPRESS=DEFLATE` in server capability responses (RFC 1951
and 4978) and, unlike [`IMAPDecodeDeflateCompression`](plugins/IMAPDecodeDeflateCompression.py), handles this entirely within the plugin, keeping an uncompressed channel to the client, but enabling compressed communication with the server.
Place this plugin at the end of the chain in the proxy's configuration file.

- [`IMAPIgnoreSentMessageUpload`](plugins/IMAPIgnoreSentMessageUpload.py): An example IMAP plugin that accepts client requests to upload sent messages to an IMAP mailbox, but silently discards them without sending to the server.
This plugin helps avoid message duplication for servers that automatically place messages sent via SMTP into the relevant IMAP mailbox.
Note that many clients are aware of this behaviour and provide an option to not upload sent messages – if this is available it is a much more efficient solution than adding a proxy plugin.

- [`SMTPBlackHole`](plugins/SMTPBlackHole.py): An example SMTP plugin that accepts incoming email messages but silently discards them without passing to the remote server.
Useful for testing email sending tools with no risk of actually delivering messages.

- [`SMTPSimpleMailingList`](plugins/SMTPSimpleMailingList.py): An example SMTP plugin that replaces specified recipient addresses in outgoing emails with one or more different recipients.
Please note that this is a relatively simplistic example of how such a plugin could work, and if you try to confuse/break it you will likely succeed (please use a real mailing list if you need to handle this).
However, this approach works well for basic cases - for example, when you often find yourself sending an email to the same addresses, you can consolidate the group into a single address and never forget any recipients.
Recipients themselves only ever see the real addresses, not your list, so there is no danger of replying to an address that does not exist.
It recommended to chain with [`SMTPBlackHole`](plugins/SMTPBlackHole.py) and enable debug mode in the proxy at first use to check intended behaviour.


## Creating your own plugins
Extend [`BasePlugin`](plugins/BasePlugin.py) to create your own plugins that customise IMAP/POP/SMTP client/server behaviour.
The two overridable methods `receive_from_client` and `receive_from_server` give access to raw IMAP/POP/SMTP messages/commands as they are received.
The return values from these overridden methods, and the two sender methods `send_to_server` and `send_to_client` allow you to send messages/commands to the client/server (and other chained plugins) in response.
The `log_debug`, `log_info` and `log_error` methods allow you to post messages to the proxy's main log.
Note that plugins are inserted after authentication has finished – you will not receive (or be able to send) anything between the client and/or server until that process is complete.
