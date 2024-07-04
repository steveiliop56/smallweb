This page will guide you through the process of setting up your local environment for smallweb on MacOS.

At the end of this process, each folder in `~/www` will be mapped to domain with a `.localhost` suffix. For example, the folder `~/www/example` will be accessible at `https://example.localhost`.

This setup is useful for developing and testing smallweb apps locally, without having to deploy them to the internet.

If you want to expose your apps to the internet instead, you can follow the [Cloudflare Tunnel setup guide](../cloudflare/tunnel.md).

## Architecture

The following diagram illustrates the architecture of the local setup:

![Localhost architecture](./architecture.excalidraw.png)

The components needed are:

- a dns server to map `.localhost` domains to `127.0.0.1` ip address (dnsmasq)
- a reverse proxy to automatically generate https certificates for each domain, and redirect traffic to the smallweb evaluation server (caddy)
- a service to map each domain to the corresponding folder in ~/www, and spawn a deno subprocess for each request (smallweb)
- a runtime to evaluate the application code (deno)

## Installation

In the future, we might provide a script to automate this process, but for now, it's a manual process.

### Install Required Dependencies

```sh
# install homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# install all required dependencies
brew install deno go caddy dnsmasq
```

### Install Smallweb

```sh
git clone https://github.com/pomdtr/smallweb
cd smallweb && go install
echo "export PATH=$PATH:$(go env GOPATH)/bin" >> ~/.zshrc
smallweb service install
```

### Setup Caddy

```sh
# Write caddy configuration
cat <<EOF > /opt/homebrew/etc/Caddyfile
*.localhost {
  tls internal {
    on_demand
  }

  reverse_proxy localhost:7777
}
EOF

# Run caddy in the background
brew services start caddy

# Add caddy https certificates to your keychain
caddy trust

mkdir -p ~/www
# Indicate to deno to use the keychain for tls certificates
echo "DENO_TLS_CA_STORE=system" >> ~/www/.env
```

### Setup dnsmasq

```sh
# Write dnsmasq configuration
echo "address=/.localhost/127.0.0.1" >> /opt/homebrew/etc/dnsmasq.conf

# Run dnsmasq in the background
sudo brew services start dnsmasq

# Indicates to the system to use dnsmasq for .localhost domains
sudo mkdir -p /etc/resolver
cat <<EOF | sudo tee -a /etc/resolver/localhost
nameserver 127.0.0.1
EOF
```

## Testing the setup

First, let's create a dummy smallweb website:

```sh
mkdir -p ~/www/example
CAT <<EOF > ~/www/example/main.ts
export default {
  fetch() {
    return new Response("Smallweb is running", {
      headers: {
        "Content-Type": "text/plain",
      },
    });
  }
}
EOF
```

If everything went well, you should be able to access `https://example.localhost` in your browser, and see the message `Smallweb is running`.