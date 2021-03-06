# escape=`

# Stage 0: prepare files & build prerequisites
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.7.2-windowsservercore-ltsc2019 AS prepare

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ARG COMMERCE_SIF_PACKAGE
ARG COMMERCE_SDK_PACKAGE
ARG SITECORE_BIZFX_PACKAGE

WORKDIR /Files

COPY files /Files
COPY scripts /Scripts

# Expand installation files
RUN Expand-Archive -Force -Path "/Files/$Env:COMMERCE_SDK_PACKAGE" -DestinationPath '/Files/Sitecore.Commerce.SDK'; `
    Expand-Archive -Force -Path "/Files/$Env:COMMERCE_SIF_PACKAGE" -DestinationPath '/Files/SIF.Sitecore.Commerce'; `
    Expand-Archive -Force -Path "/Files/$Env:SITECORE_BIZFX_PACKAGE" -DestinationPath '/Files/Sitecore.BizFX';

# Remove the long retry in DeployCommerceContent.psm1
RUN (Get-Content -path 'C:/Files/SIF.Sitecore.Commerce/Modules/DeployCommerceContent/DeployCommerceContent.psm1' -Raw) | Foreach-Object { `
    $_ -replace '-lt 30', '-lt 1' `
       -replace 'Start-Sleep -Seconds 10', '' `
    } | Set-Content 'C:/Files/SIF.Sitecore.Commerce/Modules/DeployCommerceContent/DeployCommerceContent.psm1'

# Stage 1: create actual image
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.7.2-windowsservercore-ltsc2019

ARG HOST_NAME="commerce"
ARG SITECORE_HOSTNAME="sitecore"
ARG SHOP_NAME="CommerceEngineDefaultStorefront"
ARG ENVIRONMENT_NAME="HabitatAuthoring"
ARG COMMERCE_ENGINE_PACKAGE
ARG SITECORE_IDENTITY_PACKAGE
ARG PLUMBER_FILE_NAME="plumber.zip"

ARG SQL_USER="sa"
ARG SQL_SA_PASSWORD
ARG SQL_DB_PREFIX
ARG SQL_SERVER="mssql"
ARG SOLR_PORT=8983
ARG SOLR_CORE_PREFIX="sitecore"

ARG XCONNECT_CERT_PATH
ARG SITECORE_CERT_PATH
ARG COMMERCE_CERT_PATH
ARG ROOT_CERT_PATH

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

WORKDIR /Files

# Copy required files
COPY scripts /Scripts

COPY files/*.pfx /Files/
COPY --from=prepare /Files/Sitecore.Commerce.SDK /Files/Sitecore.Commerce.SDK/
COPY --from=prepare /Files/SIF.Sitecore.Commerce /Files/SIF.Sitecore.Commerce/
COPY --from=prepare /Files/Sitecore.BizFX /Files/Sitecore.BizFX/
COPY --from=prepare /Files/${COMMERCE_ENGINE_PACKAGE} /Files/
COPY --from=prepare /Files/${SITECORE_IDENTITY_PACKAGE} /Files/
COPY --from=prepare /Files/${PLUMBER_FILE_NAME} /Files/

# Trust & import self signed certificates
RUN /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:COMMERCE_CERT_PATH -secret 'secret' -storeName 'Root' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:COMMERCE_CERT_PATH -secret 'secret' -storeName 'My' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:ROOT_CERT_PATH -secret 'secret' -storeName 'Root' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:ROOT_CERT_PATH -secret 'secret' -storeName 'My' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:SITECORE_CERT_PATH -secret 'secret' -storeName 'Root' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:XCONNECT_CERT_PATH -secret 'secret' -storeName 'Root' -storeLocation 'LocalMachine'; `
    /Scripts/Import-Certificate.ps1 -certificateFile /Files/$Env:XCONNECT_CERT_PATH -secret 'secret' -storeName 'My' -storeLocation 'LocalMachine'

RUN net user /add commerceuser 'Pa$$w0rd'; `
    Set-LocalUser -Name 'commerceuser' -PasswordNeverExpires:$true

# Install SIF
RUN /Scripts/Install-SIF.ps1

# Configure
RUN [Environment]::SetEnvironmentVariable('PSModulePath', $env:PSModulePath + ';/Files/SIF.Sitecore.Commerce/Modules'); `
    $solrUrl = 'http://solr:{0}/solr' -f $Env:SOLR_PORT; `
    $engineZip = '/Files/{0}' -f $Env:COMMERCE_ENGINE_PACKAGE; `
    Install-SitecoreConfiguration -Path '/Files/SIF.Sitecore.Commerce/Configuration/Commerce/CommerceEngine/CommerceEngine.Deploy.json' `
    -CommerceServicesDbServer $Env:SQL_SERVER `
    -CommerceServicesDbName SitecoreCommerce9_SharedEnvironments `
    -CommerceServicesGlobalDbName SitecoreCommerce9_Global `
    -CommerceServicesPostfix Sc9 `
    -SitecoreDbServer $Env:SQL_SERVER `
    -SitecoreCoreDbName "${$Env:SQL_DB_PREFIX}_Core"`
    -SolrUrl $solrUrl `
    -SearchIndexPrefix $Env:SOLR_CORE_PREFIX `
    -CommerceOpsServicesPort 5015 `
    -CommerceShopsServicesPort 5005 `
    -CommerceAuthoringServicesPort 5000 `
    -CommerceMinionsServicesPort 5010 `
    -SiteHostHeaderName 'sitecore' `
    -UserAccount @{ Domain = $Env:COMPUTERNAME; UserName = 'commerceuser'; Password = 'Pa$$w0rd' } `
    -CommerceEngineDacPac '/Files/Dacpac.dacpac' `
    -SitecoreCommerceEngineZipPath $engineZip `
    -CommerceSearchProvider 'SOLR' `
    -CertificateName 'commerce' `
    -Skip "DeployCommerceDatabase", "AddCommerceUserToCoreDatabase"

RUN [Environment]::SetEnvironmentVariable('PSModulePath', $env:PSModulePath + ';/Files/SIF.Sitecore.Commerce/Modules'); `
    Install-SitecoreConfiguration -Path '/Files/SIF.Sitecore.Commerce/Configuration/Commerce/SitecoreBizFx/SitecoreBizFx.json' `
    -SitecoreBizFxServicesContentPath './Sitecore.BizFX' `
    -CommerceAuthoringServicesPort 5000 `
    -UserAccount @{ Domain = $Env:COMPUTERNAME; UserName = 'commerceuser'; Password = 'Pa$$w0rd' }

RUN [Environment]::SetEnvironmentVariable('PSModulePath', $env:PSModulePath + ';/Files/SIF.Sitecore.Commerce/Modules'); `
    $zip = '/Files/{0}' -f $Env:SITECORE_IDENTITY_PACKAGE; `
    Install-SitecoreConfiguration -Path '/Files/SIF.Sitecore.Commerce/Configuration/Commerce/SitecoreIdentityServer/SitecoreIdentityServer.json' `
    -SitecoreIdentityServerZipPath $zip `
    -SitecoreIdentityServerName 'SitecoreIdentityServer' `
    -SitecoreDbServer $Env:SQL_SERVER `
    -SitecoreCoreDbName "${$Env:SQL_DB_PREFIX}_Core" `
    -UserAccount @{ Domain = $Env:COMPUTERNAME; UserName = 'commerceuser'; Password = 'Pa$$w0rd' }    

COPY xc/commerce/UpdateConnectionString.ps1 /Scripts

RUN /Scripts/UpdateConnectionString.ps1 -folder c:\inetpub\wwwroot\CommerceAuthoring_Sc9 `
    -userName 'sa' `
    -password $Env:SQL_SA_PASSWORD `
    -server $Env:SQL_SERVER; `
    /Scripts/UpdateConnectionString.ps1 -folder c:\inetpub\wwwroot\CommerceMinions_Sc9 `
    -userName 'sa' `
    -password $Env:SQL_SA_PASSWORD `
    -server $Env:SQL_SERVER; `
    /Scripts/UpdateConnectionString.ps1 -folder c:\inetpub\wwwroot\CommerceOps_Sc9 `
    -userName 'sa' `
    -password $Env:SQL_SA_PASSWORD `
    -server $Env:SQL_SERVER; `
    /Scripts/UpdateConnectionString.ps1 -folder c:\inetpub\wwwroot\CommerceShops_Sc9 `
    -userName 'sa' `
    -password $Env:SQL_SA_PASSWORD `
    -server $Env:SQL_SERVER

# Install hosting, IIS URL Rewrite, and vim
RUN Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')); `
    choco install -y --params="Quiet" dotnetcore-windowshosting; `
    choco install -y --params="Quiet" urlrewrite; `
    /Scripts/Install-Vim.ps1;

RUN $pathToAppSettings  = $(Join-Path -Path c:\inetpub\wwwroot\SitecoreIdentityServer\wwwroot -ChildPath "appsettings.json"); `
    $json = Get-Content $pathToAppSettings -raw | ConvertFrom-Json; `
    $connectionString = 'Data Source={0};Initial Catalog=Sitecore_Core;Integrated Security=False;User Id={1};Password={2};' -f $Env:SQL_SERVER, 'sa', $Env:SQL_SA_PASSWORD; `
    $json.AppSettings.SitecoreMembershipOptions.ConnectionString = $connectionString; `
    $json = ConvertTo-Json $json -Depth 100; `
    Set-Content $pathToAppSettings -Value $json -Encoding UTF8;

# Set the certificate details of the certificate sitecore will connect with
RUN $CommerceServicesPathCollection = @('C:\\inetpub\\wwwroot\\CommerceAuthoring_Sc9', 'C:\\inetpub\\wwwroot\\CommerceMinions_Sc9', `
                                       'C:\\inetpub\\wwwroot\\CommerceOps_Sc9', 'C:\\inetpub\\wwwroot\\CommerceShops_Sc9'); `
    $cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2; `
    $cert.Import('c:\\Files\\commerce.pfx', 'secret', 'MachineKeySet'); `
    foreach($path in $CommerceServicesPathCollection) { `
        $pathToJson = $(Join-Path -Path $path -ChildPath "wwwroot\config.json"); `
        $originalJson = Get-Content $pathToJson -Raw | ConvertFrom-Json; `
        $certificateNode = $originalJson.Certificates.Certificates[0]; `
		$certificateNode.Thumbprint = $cert.Thumbprint; `
        $appSettingsNode = $originalJson.AppSettings; `
        $appSettingsNode.AntiForgeryEnabled = $False; `
		$originalJson | ConvertTo-Json -Depth 100 -Compress | set-content $pathToJson; `
    } 

# Configure the business tools
RUN $pathToJson = $(Join-Path -Path 'C:\\inetpub\\wwwroot\\SitecoreBizFx' -ChildPath "assets\\config.json"); `
    $originalJson = Get-Content $pathToJson -Raw | ConvertFrom-Json; `
    $originalJson.EnvironmentName = $Env:ENVIRONMENT_NAME; `
    $originalJson.ShopName = $Env:SHOP_NAME; `
    $originalJson | ConvertTo-Json -Depth 100 -Compress | set-content $pathToJson;

# Install plumber
RUN Expand-Archive -Path "/Files/$Env:PLUMBER_FILE_NAME" -DestinationPath 'c:\\inetpub\\plumber'; `
    Import-Module -Name WebAdministration; `
    $iisApp = New-Item IIS:\Sites\Plumber -bindings @{protocol='http';bindingInformation='*:4000:*'} -physicalPath 'c:\inetpub\plumber'; `
    $pathToJson = $(Join-Path -Path 'C:\\inetpub\\plumber' -ChildPath "static\\config.json"); `
    $originalJson = Get-Content $pathToJson -Raw | ConvertFrom-Json; `
    $originalJson.EngineUri = ('https://{0}:5000' -f $Env:HOST_NAME); `
    $originalJson.IdentityServerUri = ('https://{0}:5050' -f $Env:HOST_NAME); `
    $originalJson.PlumberUri = ('http://{0}:4000' -f $Env:HOST_NAME); `
    $originalJson | ConvertTo-Json -Depth 100 -Compress | set-content $pathToJson; `
    $pathToAppSettings  = $(Join-Path -Path c:\inetpub\wwwroot\SitecoreIdentityServer\wwwroot -ChildPath "appsettings.json"); `
    $json = Get-Content $pathToAppSettings -raw | ConvertFrom-Json; `
    $client = @{}; `
    $client.ClientId = 'Plumber'; `
    $client.ClientName = 'Plumber'; `
    $client.AccessTokenType = 0; `
    $client.AccessTokenLifetimeInSeconds = 3600; `
    $client.IdentityTokenLifetimeInSeconds = 3600; `
    $client.AllowAccessTokensViaBrowser = $true; `
    $client.RequireConsent = $false; `
    $client.RequireClientSecret = $false; `
    $client.AllowedGrantTypes = @('implicit'); `
    $client.AllowedScopes = @('openid', 'dataEventRecords', 'dataeventrecordsscope', 'securedFiles', 'securedfilesscope', 'role', 'EngineAPI'); `
    $client.RedirectUris = @(('http://{0}:4000' -f $Env:HOST_NAME), ('http://{0}:4000/?' -f $Env:HOST_NAME)); `
    $client.PostLogoutRedirectUris = @(('http://{0}:4000' -f $Env:HOST_NAME), ('http://{0}:4000/?' -f $Env:HOST_NAME)); `
    $client.AllowedCorsOrigins = @(('http://{0}:4000' -f $Env:HOST_NAME), ('http://{0}:4000' -f $Env:HOST_NAME)); `
    $json.AppSettings.Clients += $client; `
    $json = ConvertTo-Json $json -Depth 100; `
    Set-Content $pathToAppSettings -Value $json -Encoding UTF8

# Expose Plumber port
EXPOSE 4000

COPY xc/commerce/WatchDirectoryMultiple.ps1 /Scripts
COPY xc/commerce/WatchDefaultDirectories.ps1 /Scripts
COPY xc/commerce/UpdateHostname.ps1 /Scripts
COPY xc/commerce/Boot.ps1 C:/

ENTRYPOINT ["powershell", "C:/Boot.ps1"]
CMD [ "-commerceHostname commerce.local -sitecoreHostname sitecore" ]