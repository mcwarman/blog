---
title: GitHub App Tokens in Powershell
author: Matthew
date: 2023-11-16T11:41:29+01:00

---

There are many good reason to use GitHub App tokens. The major one I've found in the context of GitHub is that using a token to commit to PR branch will ensure workflows actually get triggered, which when paired with repository that has required branch checks is very important (see: [Rise of the \(GitHub\) Bots]({{< ref "/posts/2023-10-16-rise-of-the-github-bots.md" >}})).

When in GitHub land there's `tibdex/github-app-token` which does all the heavy lifting in creating the token. But I was working on a project that needed to run on Azure DevOps and specifically on windows runners.

So it was time to move out of bash and write a script Powershell to retrieve the token. There's some good documentation on how GitHub App tokens work but at high-level it involves the following steps:

1. Create a JWT signed by app private key
2. Make an GitHub API request to retrieve token

```powershell
using namespace System.Net

function ConvertTo-Base64UrlString {
  param (
    [Parameter(Mandatory = $true)]$in
  )
  $result = [string]::Empty
  if ($in -is [string]) {
    $result = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($in))
  }
  elseif ($in -is [byte[]]) {
    $result = [Convert]::ToBase64String($in)
  }
  else {
    throw "Unsupported type $($in.GetType())"
  }

  return $result -replace '\+', '-' -replace '/', '_' -replace '='
}

$gitHubAppId = $env:GITHUB_APP_ID
$gitHubAppInstallationId = $env:GITHUB_APP_INSTALLATION_ID
$gitHubAppPrivateKey = $env:GITHUB_APP_PRIVATE_KEY

$now = (Get-Date).ToUniversalTime()
$createDate = [int64](Get-Date($now.AddSeconds(-30)) -UFormat "%s")
$expiryDate = [int64](Get-Date($now.AddMinutes(1)) -UFormat "%s")
$payloadJson = [Ordered]@{
  iss = $gitHubAppId
  iat = $createDate
  exp = $expiryDate
} | ConvertTo-Json -Compress

$payloadBase64 = ConvertTo-Base64UrlString $payloadJson

$headerJson = [Ordered]@{
  alg = "RS256"
  typ = "JWT"
} | ConvertTo-Json -Compress

$headerBase64 = ConvertTo-Base64UrlString $headerJson

$unsignedJWT = $headerBase64 + '.' + $payloadBase64

$privateKey = [System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($gitHubAppPrivateKey))

$signingObj = [Security.Cryptography.RSA]::Create()
$signingObj.ImportFromPem($privateKey.ToCharArray())

$unsignedJWTbytes = [Text.Encoding]::ASCII.GetBytes($unsignedJWT)

$signedJWTbytes = $signingObj.SignData(
  $unsignedJWTbytes,
  [Security.Cryptography.HashAlgorithmName]::SHA256,
  [Security.Cryptography.RSASignaturePadding]::Pkcs1
)

$signedJWTBase64 = ConvertTo-Base64UrlString $signedJWTbytes

$jwt = $unsignedJWT + '.' + $signedJWTBase64

$headers = [System.Collections.Generic.Dictionary[string, string]]::new()
$headers.Add("Accept", "application/vnd.github+json")
$headers.Add("X-GitHub-Api-Version", "2022-11-28")
$headers.Add("Authorization", ("Bearer {0}" -f $jwt))

$url = "https://api.github.com/app/installations/{0}/access_tokens" -f $gitHubAppInstallationId

$response = Invoke-WebRequest $url -Method 'POST' -Headers $headers

if ($response.StatusCode -ne 201) {
  throw "Unexpected HTTP Code"
}

$json = $response.Content | ConvertFrom-Json

Write-Host "##vso[task.setvariable variable=GitHubAccessToken;issecret=true]$($json.token)"
```
