---
title: 'Security | Rspamd'
---

## About

Rspamd is a ["fast, free and open-source spam filtering system"][rspamd-homepage]. DMS integrates Rspamd like any other service. We provide a very simple but easy to maintain setup of Rspamd.

If you want to have a look at the default configuration files for Rspamd that DMS packs, navigate to [`target/rspamd/` inside the repository][dms-default-configuration]. Please consult the [section "The Default Configuration"](#the-default-configuration) section down below for a written overview.

[rspamd-homepage]: https://rspamd.com/
[dms-default-configuration]: https://github.com/docker-mailserver/docker-mailserver/tree/master/target/rspamd

## Related Environment Variables

The following environment variables are related to Rspamd:

1. [`ENABLE_RSPAMD`](../environment.md#enable_rspamd)
2. [`ENABLE_RSPAMD_REDIS`](../environment.md#enable_rspamd_redis)
3. [`RSPAMD_CHECK_AUTHENTICATED`](../environment.md#rspamd_check_authenticated)
4. [`RSPAMD_GREYLISTING`](../environment.md#rspamd_greylisting)
5. [`RSPAMD_HFILTER`](../environment.md#rspamd_hfilter)
6. [`RSPAMD_HFILTER_HOSTNAME_UNKNOWN_SCORE`](../environment.md#rspamd_hfilter_hostname_unknown_score)
7. [`RSPAMD_LEARN`](../environment.md#rspamd_learn)
8. [`SPAM_SUBJECT`](../environment.md#spam_subject)
9. [`MOVE_SPAM_TO_JUNK`][docs-spam-to-junk]
10. [`MARK_SPAM_AS_READ`](../environment.md#mark_spam_as_read)

With these variables, you can enable Rspamd itself, and you can enable / disable certain features related to Rspamd.

[docs-spam-to-junk]: ../environment.md#move_spam_to_junk

## The Default Configuration

### Mode of Operation

!!! tip "Attention"

    Read this section carefully if you want to understand how Rspamd is integrated into DMS and how it works (on a surface level).

Rspamd is integrated as a milter into DMS. When enabled, Postfix's `main.cf` configuration file includes the parameter `rspamd_milter = inet:localhost:11332`, which is added to `smtpd_milters`. As a milter, Rspamd can inspect incoming and outgoing e-mails.

Each mail is assigned what Rspamd calls symbols: when an e-mail matches a specific criterion, the mail receives a symbol. Afterwards, Rspamd applies a _spam score_ (as usual with anti-spam software) to the e-mail.

- The score itself is calculated by adding the values of the individual symbols applied earlier. The higher the spam score is, the more likely the e-mail is spam.
- Symbol values can be negative (i.e., these symbols indicate the mail is legitimate, maybe because [SPF and DKIM][docs-dkim-dmarc-spf] are verified successfully) or the symbol can be positive (i.e., these symbols indicate the e-mail is spam, maybe because the e-mail contains a lot of links).

Rspamd then adds (a few) headers to the e-mail based on the spam score. Most important are `X-Spamd-Result`, which contains an overview of which symbols were applied. It could look like this:

```txt
X-Spamd-Result     default: False [-2.80 / 11.00]; R_SPF_NA(1.50)[no SPF record]; R_DKIM_ALLOW(-1.00)[example.com:s=dtag1]; DWL_DNSWL_LOW(-1.00)[example.com:dkim]; RWL_AMI_LASTHOP(-1.00)[192.0.2.42:from]; DMARC_POLICY_ALLOW(-1.00)[example.com,none]; RWL_MAILSPIKE_EXCELLENT(-0.40)[192.0.2.42:from]; FORGED_SENDER(0.30)[noreply@example.com,some-reply-address@bounce.example.com]; RCVD_IN_DNSWL_LOW(-0.10)[192.0.2.42:from]; MIME_GOOD(-0.10)[multipart/mixed,multipart/related,multipart/alternative,text/plain]; MIME_TRACE(0.00)[0:+,1:+,2:+,3:+,4:~,5:~,6:~]; RCVD_COUNT_THREE(0.00)[3]; RCPT_COUNT_ONE(0.00)[1]; REPLYTO_DN_EQ_FROM_DN(0.00)[]; ARC_NA(0.00)[]; TO_MATCH_ENVRCPT_ALL(0.00)[]; RCVD_TLS_LAST(0.00)[]; DKIM_TRACE(0.00)[example.com:+]; HAS_ATTACHMENT(0.00)[]; TO_DN_NONE(0.00)[]; FROM_NEQ_ENVFROM(0.00)[noreply@example.com,some-reply-address@bounce.example.com]; FROM_HAS_DN(0.00)[]; REPLYTO_DOM_NEQ_FROM_DOM(0.00)[]; PREVIOUSLY_DELIVERED(0.00)[receiver@anotherexample.com]; ASN(0.00)[asn:3320, ipnet:192.0.2.0/24, country:DE]; MID_RHS_MATCH_FROM(0.00)[]; MISSING_XM_UA(0.00)[]; HAS_REPLYTO(0.00)[some-reply-address@dms-reply.example.com]
```

And then there is a corresponding `X-Rspamd-Action` header, which shows the overall result and the action that is taken. In our example, it would be:

```txt
X-Rspamd-Action    no action
```

Since the score is `-2.80`, nothing will happen and the e-mail is not classified as spam. Our custom [`actions.conf`][rspamd-actions-config] defines what to do at certain scores:

1. At a score of 4, the e-mail is to be _greylisted_;
2. At a score of 6, the e-mail is _marked with a header_ (`X-Spam: Yes`);
3. At a score of 7, the e-mail will additionally have their _subject re-written_ (appending a prefix like `[SPAM]`);
4. At a score of 11, the e-mail is outright _rejected_.

---

There is more to spam analysis than meets the eye: we have not covered the [Bayes training and filters][rspamc-docs-bayes] here, nor have we talked about [Sieve rules for e-mails that are marked as spam][docs-spam-to-junk].

Even the calculation of the score with the individual symbols has been presented to you in a simplified manner. But with the knowledge from above, you're equipped to read on and use Rspamd confidently. Keep on reading to understand the integration even better - you will want to know about your anti-spam software, not only to keep the bad e-mail out, but also to make sure the good e-mail arrive properly!

[docs-dkim-dmarc-spf]: ../best-practices/dkim_dmarc_spf.md
[rspamd-actions-config]: https://github.com/docker-mailserver/docker-mailserver/blob/master/target/rspamd/local.d/actions.conf
[rspamc-docs-bayes]: https://rspamd.com/doc/configuration/statistic.html

### Workers

The proxy worker operates in [self-scan mode][rspamd-docs-proxy-self-scan-mode]. This simplifies the setup as we do not require a normal worker. You can easily change this though by [overriding the configuration by DMS](#providing-custom-settings-overriding-settings).

DMS does not set a default password for the controller worker. You may want to do that yourself. In setups where you already have an authentication provider in front of the Rspamd webpage, you may want to [set the `secure_ip ` option to `"0.0.0.0/0"` for the controller worker](#with-the-help-of-a-custom-file) to disable password authentication inside Rspamd completely.

[rspamd-docs-proxy-self-scan-mode]: https://rspamd.com/doc/workers/rspamd_proxy.html#self-scan-mode

### Persistence with Redis

When Rspamd is enabled, we implicitly also start an instance of Redis in the container:

- Redis is configured to persist its data via RDB snapshots to disk in the directory `/var/lib/redis` (_or the [`/var/mail-state/`][docs::dms-volumes-state] volume when present_).
- With the volume mount the snapshot will restore the Redis data across container restarts, and provide a way to keep backup.

Redis uses `/etc/redis/redis.conf` for configuration:

- We adjust this file when enabling the internal Redis service.
- If you have an external instance of Redis to use, the internal Redis service can be opt-out via setting the ENV [`ENABLE_RSPAMD_REDIS=0`](../environment.md#enable_rspamd_redis) (_link also details required changes to the DMS Rspamd config_).

### Web Interface

Rspamd provides a [web interface][rspamc-docs-web-interface], which contains statistics and data Rspamd collects. The interface is enabled by default and reachable on port 11334.

![Rspamd Web Interface](https://rspamd.com/img/webui.png)

[rspamc-docs-web-interface]: https://rspamd.com/webui/

### DNS

DMS does not supply custom values for DNS servers to Rspamd. If you need to use custom DNS servers, which could be required when using [DNS-based black/whitelists](#rbls-realtime-blacklists-dnsbls-dns-based-blacklists), you need to adjust [`options.inc`][rspamd-docs-basic-options] yourself.

!!! tip "Making DNS Servers Configurable"

    If you want to see an environment variable (like `RSPAMD_DNS_SERVERS`) to support custom DNS servers for Rspamd being added to DMS, please raise a feature request issue.

!!! danger

    While we do not provide values for custom DNS servers by default, we set `soft_reject_on_timeout = true;` by default. This setting will cause a soft reject if a task (presumably a DNS request) timeout takes place.

    This setting is enabled to not allow spam to proceed just because DNS requests did not succeed. It could deny legitimate e-mails to pass though too in case your DNS setup is incorrect or not functioning properly.

### Logs

You can find the Rspamd logs at `/var/log/mail/rspamd.log`, and the corresponding logs for [Redis](#persistence-with-redis), if it is enabled, at `/var/log/supervisor/rspamd-redis.log`. We recommend inspecting these logs (with `docker exec -it <CONTAINER NAME> less /var/log/mail/rspamd.log`) in case Rspamd does not work as expected.

### Modules

You can find a list of all Rspamd modules [on their website][rspamd-docs-modules].

[rspamd-docs-modules]: https://rspamd.com/doc/modules/

#### Disabled By Default

DMS disables certain modules (`clickhouse`, `elastic`, `neural`, `reputation`, `spamassassin`, `url_redirector`, `metric_exporter`) by default. We believe these are not required in a standard setup, and they would otherwise needlessly use system resources.

#### Anti-Virus (ClamAV)

You can choose to enable ClamAV, and Rspamd will then use it to check for viruses. Just set the environment variable `ENABLE_CLAMAV=1`.

#### RBLs (Real-time Blacklists) / DNSBLs (DNS-based Blacklists)

The [RBL module](https://rspamd.com/doc/modules/rbl.html) is enabled by default. As a consequence, Rspamd will perform DNS lookups to a variety of blacklists. Whether an RBL or a DNSBL is queried depends on where the domain name was obtained: RBL servers are queried with IP addresses extracted from message headers, DNSBL server are queried with domains and IP addresses extracted from the message body \[[source][rbl-vs-dnsbl]\].

!!! danger "Rspamd and DNS Block Lists"

    When the RBL module is enabled, Rspamd will do a variety of DNS requests to (amongst other things) DNSBLs. There are a variety of issues involved when using DNSBLs. Rspamd will try to mitigate some of them by properly evaluating all return codes. This evaluation is a best effort though, so if the DNSBL operators change or add return codes, it may take a while for Rspamd to adjust as well.

    If you want to use DNSBLs, **try to use your own DNS resolver** and make sure it is set up correctly, i.e. it should be a non-public & **recursive** resolver. Otherwise, you might not be able ([see this Spamhaus post](https://www.spamhaus.org/faq/section/DNSBL%20Usage#365)) to make use of the block lists.

[rbl-vs-dnsbl]: https://forum.eset.com/topic/25277-dnsbl-vs-rbl-mail-security/?do=findComment&comment=119818

## Providing Custom Settings & Overriding Settings

DMS brings sane default settings for Rspamd. They are located at `/etc/rspamd/local.d/` inside the container (or `target/rspamd/local.d/` in the repository).

### Manually

!!! question "What is [`docker-data/dms/config/`][docs::dms-volumes-config]?"

If you want to overwrite the default settings and / or provide your own settings, you can place files at `docker-data/dms/config/rspamd/override.d/`. Files from this directory are copied to `/etc/rspamd/override.d/` during startup. These files [forcibly override][rspamd-docs-override-dir] Rspamd and DMS default settings.

!!! question "What is the [`local.d` directory and how does it compare to `override.d`][rspamd-docs-config-directories]?"

!!! warning "Clashing Overrides"

    Note that when also [using the `custom-commands.conf` file](#with-the-help-of-a-custom-file), files in `override.d` may be overwritten in case you adjust them manually and with the help of the file.

[rspamd-docs-override-dir]: https://www.rspamd.com/doc/faq.html#what-are-the-locald-and-overrided-directories
[rspamd-docs-config-directories]: https://rspamd.com/doc/faq.html#what-are-the-locald-and-overrided-directories

### With the Help of a Custom File

DMS provides the ability to do simple adjustments to Rspamd modules with the help of a single file. Just place a file called `custom-commands.conf` into `docker-data/dms/config/rspamd/`. If this file is present, DMS will evaluate it. The structure is _very_ simple. Each line in the file looks like this:

```txt
COMMAND ARGUMENT1 ARGUMENT2 ARGUMENT3
```

where `COMMAND` can be:

1. `disable-module`: disables the module with name `ARGUMENT1`
2. `enable-module`: explicitly enables the module with name `ARGUMENT1`
3. `set-option-for-module`: sets the value for option `ARGUMENT2` to `ARGUMENT3` inside module `ARGUMENT1`
4. `set-option-for-controller`: set the value of option `ARGUMENT1` to `ARGUMENT2` for the controller worker
5. `set-option-for-proxy`: set the value of option `ARGUMENT1` to `ARGUMENT2` for the proxy worker
6. `set-common-option`: set the option `ARGUMENT1` that [defines basic Rspamd behavior][rspamd-docs-basic-options] to value `ARGUMENT2`
7. `add-line`: this will add the complete line after `ARGUMENT1` (with all characters) to the file `/etc/rspamd/override.d/<ARGUMENT1>`

!!! example "An Example Is [Shown Down Below](#adjusting-and-extending-the-very-basic-configuration)"

!!! note "File Names & Extensions"

    For command 1 - 3, we append the `.conf` suffix to the module name to get the correct file name automatically. For commands 4 - 6, the file name is fixed (you don't even need to provide it). For command 7, you will need to provide the whole file name (including the suffix) yourself!

You can also have comments (the line starts with `#`) and blank lines in `custom-commands.conf` - they are properly handled and not evaluated.

!!! tip "Adjusting Modules This Way"

    These simple commands are meant to give users the ability to _easily_ alter modules and their options. As a consequence, they are not powerful enough to enable multi-line adjustments. If you need to do something more complex, we advise to do that [manually](#manually)!

[rspamd-docs-basic-options]: https://rspamd.com/doc/configuration/options.html

## Examples & Advanced Configuration

### A Very Basic Configuration

You want to start using Rspamd? Rspamd is disabled by default, so you need to set the following environment variables:

```env
ENABLE_RSPAMD=1
ENABLE_OPENDKIM=0
ENABLE_OPENDMARC=0
ENABLE_POLICYD_SPF=0
ENABLE_AMAVIS=0
ENABLE_SPAMASSASSIN=0
```

This will enable Rspamd and disable services you don't need when using Rspamd.

### Adjusting and Extending The Very Basic Configuration

Rspamd is running, but you want or need to adjust it? First, create a file named `custom-commands.conf` under `docker-data/dms/config/rspamd` (which translates to `/tmp/docker-mailserver/rspamd/` inside the container). Then add you changes:

1. Say you want to be able to easily look at the frontend Rspamd provides on port 11334 (default) without the need to enter a password (maybe because you already provide authorization and authentication). You will need to adjust the controller worker: `set-option-for-controller secure_ip "0.0.0.0/0"`.
2. You additionally want to enable the auto-spam-learning for the Bayes module? No problem: `set-option-for-module classifier-bayes autolearn true`.
3. But the chartable module gets on your nerves? Easy: `disable-module chartable`.

??? example "What Does the Result Look Like?"
    Here is what the file looks like in the end:

    ```bash
    # See 1.
    # ATTENTION: this disables authentication on the website - make sure you know what you're doing!
    set-option-for-controller secure_ip "0.0.0.0/0"

    # See 2.
    set-option-for-module classifier-bayes autolearn true

    # See 3.
    disable-module chartable
    ```

### DKIM Signing

There is a dedicated [section for setting up DKIM with Rspamd in our documentation][docs-dkim-with-rspamd].

[docs-dkim-with-rspamd]: ../best-practices/dkim_dmarc_spf.md#dkim

### _Abusix_ Integration

This subsection gives information about the integration of [Abusix], "a set of blocklists that work as an additional email security layer for your existing mail environment". The setup is straight-forward and well documented:

1. [Create an account](https://app.abusix.com/signup)
2. Retrieve your API key
3. Navigate to the ["Getting Started" documentation for Rspamd][abusix-rspamd-integration] and follow the steps described there
4. Make sure to change `<APIKEY>` to your private API key

We recommend mounting the files directly into the container, as they are rather big and not manageable with the [modules script](#with-the-help-of-a-custom-file). If mounted to the correct location, Rspamd will automatically pick them up.

While _Abusix_ can be integrated into Postfix, Postscreen and a multitude of other software, we recommend integrating _Abusix_ only into a single piece of software running in your mail server - everything else would be excessive and wasting queries. Moreover, we recommend the integration into suitable filtering software and not Postfix itself, as software like Postscreen or Rspamd can properly evaluate the return codes and other configuration.

[Abusix]: https://abusix.com/
[abusix-rspamd-integration]: https://docs.abusix.com/abusix-mail-intelligence/gbG8EcJ3x3fSUv8cMZLiwA/getting-started/dmw9dcwSGSNQiLTssFAnBW#rspamd

[docs::dms-volumes-config]: ../advanced/optional-config.md#volumes-config
[docs::dms-volumes-state]: ../advanced/optional-config.md#volumes-state
