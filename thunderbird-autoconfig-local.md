# Thunderbird Local Autoconfig - When You Don't Control the Domain

## The Problem

You're connecting another account to Thunderbird but it fails to auto-detect settings for the domain. You have to manually enter IMAP server settings such as server address, port, and encryption config—and then do the same for SMTP.
You repeat this for every account. It's tedious and time-consuming. There's a simpler way.

You can automate this using autoconfig. Options include DNS configuration, a file hosted on the domain's server, or a **local file**. Any of these will tell Thunderbird how to configure IMAP and SMTP for a particular domain, so you only need to enter the email address and password—everything else is configured automatically.

Thunderbird can automatically detect proper IMAP and SMTP server settings for a domain, but it pulls this information from DNS, the domain's web server, or Mozilla's ISP database.
But what if you don't have access to the domain's DNS or server?

## The Solution

Thunderbird checks **local configuration files first** before trying remote autoconfig. You can place XML config files directly in Thunderbird's installation directory.
It's simple, straightforward, and takes about a minute to implement. You can distribute this file to other workstations using RMM software or other deployment methods.

## File Location

Look for the `isp` folder in Thunderbird's installation directory. If it doesn't exist, create it, then place your XML file there.

### Windows
```
C:\Program Files\Mozilla Thunderbird\isp\mydomain.com.xml
```

### Linux
```
/usr/lib/thunderbird/isp/mydomain.com.xml
```
or
```
/opt/thunderbird/isp/mydomain.com.xml
```

### macOS
```
/Applications/Thunderbird.app/Contents/Resources/isp/mydomain.com.xml
```

> **Important:** The filename must exactly match the domain (the part after @ in email addresses), with the `.xml` extension.

For example, if your domain is `mydomain.com`, create a file named exactly `mydomain.com.xml` inside the `isp` folder.

## Configuration Template

Save this as `mydomain.com.xml` (replace with your actual domain):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<clientConfig version="1.1">
	<emailProvider id="myhosting">
		<domain>mydomain.com</domain>
		<incomingServer type="imap">
			<hostname>imap.myhosting.com</hostname>
			<port>993</port>
			<socketType>SSL</socketType>
			<authentication>password-cleartext</authentication>
			<username>%EMAILADDRESS%</username>
		</incomingServer>
		<outgoingServer type="smtp">
			<hostname>smtp.myhosting.com</hostname>
			<port>587</port>
			<socketType>STARTTLS</socketType>
			<authentication>password-cleartext</authentication>
			<username>%EMAILADDRESS%</username>
		</outgoingServer>
	</emailProvider>
</clientConfig>
```

Some notes:
- **emailProvider id** - You can use any name here; feel free to leave it as "myhosting".
- You can use **only one** emailProvider block. If you create more than one, Thunderbird will use only the first and ignore the rest.
- In most cases, you'll only need to edit these fields: `domain`, `hostname`, `port`, and `socketType`.
- **Port numbers**: IMAP uses 993 (SSL) or 143 (STARTTLS); SMTP uses 465 (SSL) or 587 (STARTTLS); POP3 uses 995 (SSL) or 110 (STARTTLS).
- The most common configuration is 993 for IMAP (SSL) and 587 for SMTP (STARTTLS).
- For **socketType**, always use the protocol that corresponds to the port number—for example, SSL for port 993.

## How It Works

Thunderbird checks configuration sources in this order:

1. **Local file** `installdir/isp/domain.xml` ← Your config here
2. `http://autoconfig.domain.com/mail/config-v1.1.xml`
3. `http://domain.com/.well-known/autoconfig/mail/config-v1.1.xml`
4. Mozilla ISPDB database
5. Common server name guessing
6. Manual configuration

Since local files are checked **first**, your configuration takes priority.

## Result

After placing the XML file:

1. Restart Thunderbird
2. Add a new account
3. Enter: Display Name, Email Address, Password
4. Click Continue → Thunderbird auto-detects settings
5. Done

No manual server configuration needed.

## Multiple Domains

For multiple domains using the same mail server, you have two options:

**Option A:** Create separate files for each domain
- `domain1.com.xml`
- `domain2.com.xml`

**Option B:** Add multiple `<domain>` tags in one file
```xml
<emailProvider id="myhosting">
	<domain>domain1.com</domain>
	<domain>domain2.com</domain>
	<domain>domain3.com</domain>
	<!-- rest of configuration -->
</emailProvider>
```

## Notes

- Requires admin/root privileges to write to Thunderbird's installation directory
- Works entirely offline—no network requests needed
- Survives Thunderbird updates (the folder persists)

## References

- [Mozilla Autoconfiguration Wiki](https://wiki.mozilla.org/Thunderbird:Autoconfiguration)
- [Config File Format](https://wiki.mozilla.org/Thunderbird:Autoconfiguration:ConfigFileFormat)
