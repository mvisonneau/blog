+++
title = "Secure Kubernetes configuration at rest"
description = "I found that storing sensitive values for helm releases was not trivial, this is my experience around finding a solution for it."
tags = [
  "golang",
  "helm",
  "kubernetes",
  "security",
  "vault"
]
date = "2018-07-13"
categories = [
  "golang",
  "helm",
  "kubernetes",
  "security",
  "vault"
]
highlight = "true"
+++

# Kubernetes-all-the-things

If you are getting started with [Kubernetes](https://kubernetes.io/). There is one thing that will eventually happen at some point:

> How can I run all my things at scale **securely**? 🤔

Some might say:

> *Bruh! Kubernetes is secure by design, what are you talking about?!*

> *Just use a [secret resource](https://kubernetes.io/docs/concepts/configuration/secret/)!*

Which is actually correct! You can simply do something like this :

```
~$ kubectl create secret generic mysecret --from-literal=key=sensitive_value
```

With sufficient permissions, you should be able to access the resource and read deciphered values.

```
~$ kubectl get secret mysecret -o json | jq -r .data.key | base64 --decode
sensitive_value
```

**However!** This doesn't mean that the value has actually been safely stored. Kubernetes store most of the things within the `etcd` cluster it works with such as the `Secret` resources. With direct access onto it, we can actually check if the data has been ciphered or not:

```
~$ ETCDCTL_API=3 etcdctl get /registry/secrets/default/mysecret | jq -r .data.key | base64 --decode
sensitive_value
```

😱 the sensitive value is stored in plaintext!

## No panic, there is ~~an app~~ a Kubernetes resource for that!

Indeed, in order to encrypt it, there is a Kubernetes resource called [EncryptionConfig](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/). If you are not familiar with it I would strongly recommend to have a look into it. It allows you to choose how you want to encrypt *things* on your cluster.

It is very flexible and easy to configure. Let's start with using a salt that we will manage and rotate ourselves. If you prefer, have a look onto the documentation and use the other type of providers such as the cloud provided KMS ones for instance.

```
~$ cat <<EOF > /etc/kubernetes/encryption-config.yaml
---
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - secretbox:
          keys:
            - name: key1
              secret: kwQZ1LLvqInL8oUBEMx7UL6jl+Yh07ry4fqMIKQn+KA= # A 32 byte random string base64 encoded
      - identity: {}
EOF
```

You then need to restart your [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) after adding the following flag:

```
--experimental-encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

Now that we have enabled encryption, we can trigger a rewrite of all the secrets that we already have stored to secure them (even though they are compromised 😅):

```
~$ kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Having a second look at `etcd`, we should now be able to see that our values are safely stored.

```
~$ ETCDCTL_API=3 etcdctl get /registry/secrets/default/mysecret
<ciphered_text>
```

All set! \o/

But we are not done yet..

## Infrastructure as Code

> Launching stacks on Kubernetes is one thing, organising and running them at scale is something else.

The first thing that you are going to attempt (hopefully 🙏) is to treat your configuration `as code`. This means that you are going to store it somewhere/somehow, whether through a [VCS](https://en.wikipedia.org/wiki/VCS) or using your own mechanisms. Treating those configs as regular code empowers us with every practice we might already have in place for software development

I personally chose to store all my configuration files in [git](https://git-scm.com/) (which is apparently being argued if it is a VCS or not)

On top of it, we have CI pipelines in order to automatically push the changes onto the clusters after and during peer reviewing through merge requests.

If we look back at the example we did earlier, we could refactor it to something like this:

```bash
~$ git init
~$ cat <<EOF > secret.yml
---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  sensitive: dmFsdWU=
EOF
~$ git add secret.yml
~$ git commit -m"I have no idea what I'm doing \o/ :)"
```

As you would have figured out by now, we have an issue here. As safe as the secret resource will be once stored in the Kubernetes cluster, I'm basically storing it in plaintext within my git repository which is definitely not acceptable.

#### Let's encrypt! (at rest)

Many solutions are available, here is a few example :

- Use a [GPG](https://www.gnupg.org/gph/en/manual/x110.html) key to store them
- Store them into protected CI environment variables and substitute them at runtime eg: [GitLab CI](https://docs.gitlab.com/ce/ci/variables/#protected-variables)
- [hiera-eyaml](https://puppet.com/blog/encrypt-your-data-using-hiera-eyaml)
- [ansible-vault](https://docs.ansible.com/ansible/2.5/user_guide/vault.html)
- [openssl](https://www.devco.net/archives/2006/02/13/public_-_private_key_encryption_using_openssl.php) with a public key
- [AWS](https://aws.amazon.com/kms‎) or [GCP](https://cloud.google.com/kms/) KMS
- and many others that are not on this non-exhaustive list!

But I did not find exactly what I was looking for in all those methods. I liked the way `hiera-eyaml` was conceived and working. I was also already using and a big fan of [Hashicorp Vault](https://vaultproject.io/). Therefore I figured there could be something to do with that. Hence [s5](https://github.com/mvisonneau/s5)!

> The idea was to be able to encrypt **only what's necessary** within **any kind of file**. Using `Vault` as the backend for managing the encryption key *(aka. transit key)* as well as the permissions around it.

I tried to keep the concept, implementation and usage as trivial as possible. This is what you can expect in terms of overhead to your commands when using it :

```bash
~$ s5 render secret.yml | kubectl apply -f -
```

And this is what your `secret.yml` would now look like:

```
---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  sensitive: {{ s5:EtWnJ8ZyuwzRn8I3jw== }}
```

`s5` will basically replace the following **regexp** as many times as it finds it within your file :

`{{ s5:.* }}`

🎉 you can now commit or share your file to whomever you want (or not 😅)!

> What I like about it is that it makes it easier to **review changes made to secrets** as we **do not cipher the entire file**.

> Plus, having `Vault` as the encryption backend makes it **easier to handle who can get ciphering and deciphering accesses** as well as cloud provider agnostic (on top of being open source 💚)


#### Pro-tip

Most of the engineers working on K8S can easily have ciphering access, whilst we reserve the deciphering accesses for secure CI pipelines.

## Ok cool, so how does this works and how do I get started ?

Most of the documentation (so far) can be found directly onto the project [README](https://github.com/mvisonneau/s5/blob/master/README.md).
However here is a quick getting started guide!

### First, you need to have an operational **Vault** endpoint!

You will also need permissions to manipulate the transit backend or already have cipher/decipher access onto a existent transit key otherwise.

> 😅 seriously, I don't have time for that, I just to try it out..

Well, having a Vault endpoint ready is only **about 4 commands** that you can run on your laptop on most common OSes! [Full download page can be found here](https://www.vaultproject.io/downloads.html).

```
~$ wget https://releases.hashicorp.com/vault/0.10.3/vault_0.10.3_linux_amd64.zip -O /tmp/vault.zip
~$ sudo unzip /tmp/vault.zip -d /usr/local/bin
~$ sudo chmod +x /usr/local/bin/vault
~$ vault server -dev
```

or if like me you are a bit preservative with your laptop and like to run everything in containers it's a **one liner**:

```
~$ docker run -it --rm -p 8200:8200 vault:latest server -dev -dev-listen-address=0.0.0.0
```

### Let's now start encrypting stuff!

Now that the Vault endpoint is ready, for ease of use you can configure some environment variables. You can find the values to fill in from the output of the `vault server -dev` command.

```
~$ export VAULT_ADDR=http://127.0.0.1:8200
~$ export VAULT_TOKEN=268c7fc2-ff96-c11d-150b-074b009f2865 # For this one refer to the root token
```

👆 You would have noticed that it listens over `HTTP`, this is because we are running a *development* environment locally. Please don't be that guy that runs it like that in *production* 🙏!

You can check that everything is running well by doing a `vault status`.

```
~$ vault status -format json
{
  "type": "shamir",
  "sealed": false,
  "t": 1,
  "n": 1,
  "progress": 0,
  "nonce": "",
  "version": "0.10.1",
  "cluster_name": "vault-cluster-a20328ee",
  "cluster_id": "c2fe413d-a8c1-7008-5358-c993d5ac65f4",
  "recovery_seal": false
}
```

Now that you are ready, you need to enable the `transit backend`:

```
~$ vault secrets enable transit
Success! Enabled the transit secrets engine at: transit/
```

`transit/` is the default path but you can feel free to mount it wherever you want.

### It's now time to get s5!

Hopefully it's exactly the same as for `Vault`:

```
# From GitHub.com
~$ wget https://github.com/mvisonneau/s5/releases/download/0.1.1/s5_linux_amd64 -O /usr/local/bin/s5
~$ sudo chmod +x /usr/local/bin/s5

# Using docker with a cute alias 💚
~$ alias s5="docker run -it --rm -v $(pwd):$(pwd) mvisonneau/s5:latest"
```

You should now be able to cipher and decipher text using the s5 cli:

```
# CIPHER
~$ s5 cipher "foobar"
{{ s5:51HqRqey8aT5AKbOXF2QPI7YvA8xY2OvRb9QrfvYpQ== }}

# DECIPHER
~$ s5 decipher "{{ s5:51HqRqey8aT5AKbOXF2QPI7YvA8xY2OvRb9QrfvYpQ== }}"
foobar
```

> What! I did not even specify any key ? 🤔

`s5` uses `default` as the default *transit key*. On top of that, if a transit key do not exist, `Vault` will create it on the **ciphering request** as long as you have sufficient privileges on the endpoint. You can easily verify that:

```
~$ vault list -format json transit/keys
[
  "default"
]
```

From now it is fairly straightforward and pretty much up to you, you can use the pattern anywhere in your files. As I mentioned earlier:

```
# CIPHER YOUR VALUES
~$ s5 cipher foo
{{ s5:t9RZefdJ38sgosufeLRPNhUx7E0lC0tIiXp4iL676Q== }}
~$ s5 cipher $( echo bar | base64 )
{{ s5:bNL+8uNkkv52xaUeH2Ty4+3f7YJKgyOTVcgpvvf1BtKGep4m }}

# WRITE THEM TO YOUR FILES
~$ cat <<EOF > secret.yml
---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  {{ s5:t9RZefdJ38sgosufeLRPNhUx7E0lC0tIiXp4iL676Q== }}: {{ s5:bNL+8uNkkv52xaUeH2Ty4+3f7YJKgyOTVcgpvvf1BtKGep4m }}
EOF

# GET THEM ON KUBERNETES
~$ s5 render secret.yml | kubectl apply -f -
secret "mysecret" created

# VERIFY DATA
~$ kubectl get secret mysecret -o json | jq -r .data.foo | base64 --decode
bar
```

## I want to know even more!

If I didn't bored you enough already, here is some other advice on how you could make a use of it at scale. I am personally a big fan of [helm](https://helm.sh/) which is as they said : *"The package manager for Kubernetes"* I would definitely recommend to have a look onto it if you are looking into ways of managing your Kubernetes stacks at scale.

This is actually how I use it myself, there is a special `--in-place/-i` flag that allows you to render the file *in-place* before letting helm consume it. This is something that is being done as part of a Makefile within a CI job.

```
~$ s5 render -i values.yml
~$ helm deploy release chart -f values.yml
```

## Even further?!

I recently started to work onto an [atom](https://atom.io) [plugin](https://github.com/mvisonneau/atom-s5) in order to easily make changes onto values. This is still a work in progress in terms of features but it works! PR are always welcome!

<gif>

## I am done!

I hope this would have been constructive. If you have questions or concerns feel free to ask, comment or to [open issues on the project](https://github.com/mvisonneau/s5/issues).

I forgot to mention that [s5](https://github.com/mvisonneau/s5) is also open-source under *Apache 2.0 licensing* so feel free to do whatever you want with it. It is also as you might have seen not bound to only Kubernetes uses cases. You can use it for any other purposes as it should support most kind of file formats.

And many thanks for getting to the end of it!
