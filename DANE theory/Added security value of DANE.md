Authentication is to prevent MITM attacks.

DANE operates in the realm between DNS and server, and authenticates the identities of these servers.
Currently (without DANE), we assume that a server responding to a client’s requests is the correct one.
We also assume that the TLS certificate the client receives is legitimately issued for this server.
But these assumptions have historically been violated.
