---
layout: single
title:  "Create a FakeCompany with Powershell"
last_modified_at: 2020-12-07
header:
  teaser: "/assets/images/posts/2019-06-02-powershell-create-fakecompany/logo-fakecompany-444x240.png"
  og_image: "/assets/images/posts/2019-06-02-powershell-create-fakecompany/logo-fakecompany-444x240.png"
type: posts
classes: wide
comments: true
read_time: true
categories:
  - Powershell
tags:
  - Windows
  - Windows Server
  - Microsoft
  - AD
  - Active Directory
  - PS
  - Powershell
  - User
  - Group
  - OU
  - Organizational Unit
  - Computer
  - CSV
  - Comma-Separated Values
  - Company
  - Fake
  - FakeCompany
  - Powershell Module
  - Module
  - Github
  - Repository
  - Repo
---

<p style="text-align: justify;"><img src="{{ site.baseurl }}/assets/images/posts/2019-06-02-powershell-create-fakecompany/logo-fakecompany-222x150.png" class="align-left">
While fessing tests in my lab, I got tired of creating users User01 groups, etc ... So I decided to create a script Powershell to create a similar structure to that of a company, allowing me to automatically create OUs, users, groups, computers, from a CSV file.</p>

## Company scheme
<img src="{{ site.baseurl }}/assets/images/posts/2019-06-02-powershell-create-fakecompany/FakeCompany_Schema.jpg" class="align-center">


## Working

<p style="text-align: justify;"><strong>The data :</strong> The data is stored in a CSV file called <strong>new_FakeCompany.csv</strong>. Each field is named as follows : field01,field02...</p>

<p style="text-align: justify;"><strong>The function :</strong> The function for creating the structure is in the <strong>FakeCompany.ps1</strong> file. This function is based on a switch based on the first field of the CSV file.</p>

```powershell
switch ($Content.('field01')) {
    'OU' {## Start create OU ##; Break }
    'Groups' { ## Start create Groups ##; Break }
    'Users' { ## Start create Users ##; Break }
    'Computers' {## Start create Computers ##; Break }
    Default {Write-Host "The file format is invalid"}
}
```

<p style="text-align: justify;">To call the function, just specify with the argument <strong>-CSVFile</strong> the location of the CSV file.</p>

```powershell
PS C:\> FakeCompany -CSVFile C:\new_FakeCompany.csv
```

## FakeCompany

```powershell
Function FakeCompany
{
<#
.SYNOPSIS
    This function allows you to create Active Directory OU, Groups, Users, Computers from CSV file.
.DESCRIPTION
    This function allows you to create Active Directory OU, Groups, Users, Computers from CSV file.
.PARAMETER FilePath
    Specify the path of a CSV file containing Active Directory OU, Groups, Users, Computers informations.
.EXAMPLE
    PS C:\> FakeCompany -CSVFile C:\new_FakeCompany.csv
.NOTES
    Valentin LEPINE
    Email : vlepineadm@outlook.com
    Twitter : @vlepineadm
    Github : https://github.com/vlepineadm/FakeCompany
#>

    [CmdletBinding()]
    [OutputType( [System.Object] )]
    PARAM
    (
        [System.String]$CSVFile
    )

    ## Global variables ##

    # Global Groups variables #
    $GroupCategory = "Security"
    $GroupScope = "Global"

    # Global Users variables #
    $LocalDomain = "corp.priv"
    $ExternalDomain = "corporate.com"
    $Password = 'P@ssW0rd!'

    ## Global Computers variables ##
    $Domain = $LocalDomain
    $OperatingSystem = "Windows 10 Enterprise"
    $OperatingSystemVersion = "10.0 (17763)"


    ## Function ##
    function Remove-StringLatinCharacters
    {
    PARAM ([string]$String)
        [Text.Encoding]::ASCII.GetString([Text.Encoding]::GetEncoding("Cyrillic").GetBytes($String))
    }

    function FormattingVar
    {
        [CmdletBinding()]
        [OutputType( [System.Object] )]
        PARAM
        (
            [System.string]$String
        )
            $String = Remove-StringLatinCharacters $String
            $String = $String.replace(' ','')
            $String = $String.ToLower()
            return ,$String
    }


    # Active Directory module import
    Try
    {
        Import-Module ActiveDirectory
    }
    Catch [FileNotFoundException]
    {
        Write-Error "The Active Directory module could not be loaded"
    }


    If ((Test-Path $CSVFile) -eq $True)
    {
        $Contents = Import-Csv -Path "$CSVFile"
        Write-Host "Launching the FakeCompanyOU function"

        foreach ($Content in $Contents)
        {
            switch ($Content.('field01')) {
                 ## Start create OU ##
                'OU' {
                    ## Name ###
                    $Name = $($Content.field02)

                    Write-Verbose $Name
                    Write-Verbose $($Content.field03)

                    ## If the OU exist
                    $ADOUExist = $(try {Get-ADOrganizationalUnit -Filter {Name -like $Name} -SearchBase "DC=corp,DC=priv"} catch {$null})
                    If ($ADOUExist)
                    {
                        Write-Host "The OU $Name already exists"
                    }
                    else
                    {
                        New-ADOrganizationalUnit -Name $Name -Path "$($Content.field03)" -PassThru -ProtectedFromAccidentalDeletion $false

                        Write-Host "The OU $Name was created"
                    }
                ; Break }
                ## End create OU ##

                ## Start create Groups ##
                'Groups' {
                    ## Name ##
                    $Name = $($Content.field02)

                    ## DisplayName ##
                    $DisplayName = $($Content.field02)

                    Write-Verbose $Name ## GRP_DPT_JUR
                    Write-Verbose $DisplayName ## GRP_DPT_JUR
                    Write-Verbose $($Content.field03) ## Département Juridique
                    Write-Verbose $($Content.field04) ## OU=Groups,OU=Paris,OU=Sites,OU=CORP,DC=corp,DC=priv

                    ## If the Group exist
                    $ADGroupExist = $(try {Get-ADGroup $Name} catch {$null})
                    If ($ADGroupExist)
                    {
                        Write-Host "The group $Name already exists"
                    }
                    else
                    {
                        New-ADGroup -Name $Name `
                        -DisplayName $DisplayName `
                        -Description $($Content.field03) `
                        -Path $($Content.field04) `
                        -GroupCategory $GroupCategory `
                        -GroupScope $GroupScope

                        Write-Host "The group $Name was created"
                    }

                ; Break }
                ## End create Groups ##

                ## Start create Users ##
                'Users' {
                    ## GivenName ##
                    $GivenName = $Content.field02.substring(0,1).toupper()+$Content.field02.substring(1).tolower()

                    ## Surname ##
                    $SurName = $Content.field03.ToUpper()

                    ## SamAccountName ##
                    $GivenNameFormat = FormattingVar $GivenName
                    $SurNameFormat = FormattingVar $SurName
                    $SamAccountName = $GivenNameFormat.ToLower()+"."+$SurNameFormat.ToLower()

                    ## Name ##
                    $Name = $SurName+" "+$GivenName+" ("+$SamAccountName+")"

                    ## Password ##
                    $Password = ConvertTo-SecureString -AsPlainText $Password -force

                    ## Email ##
                    $Email = $SamAccountName+"@"+$ExternalDomain

                    ## UserPrincipalName ##
                    $UserPrincipalName = $SamAccountName+"@"+$LocalDomain

                    ## DisplayName ##
                    $DisplayName = $Content.field03+" "+$Content.field02

                    ## Initials ##
                    $Initials = $Content.field03.substring(0,1).toupper()+$Content.field02.substring(0,1).toupper()

                    ## department group ##
                    $DepartmentGroup = $Content.field16

                    ## Business group ##
                    $BusinessGroup = $Content.field17


                    Write-Verbose $Name ## DOE Jane
                    Write-Verbose $GivenName ## Jane
                    Write-Verbose $SurName ## DOE
                    Write-Verbose $Content.field04 ## OU=Paris,OU=Sites,OU=CORP,DC=corp,DC=priv
                    Write-Verbose $Password ## System.Security.SecureString
                    Write-Verbose $Email ## jane.doe@corporate.com
                    Write-Verbose $SamAccountName ## jane.doe
                    Write-Verbose $UserPrincipalName ## DOE Jane
                    Write-Verbose $DisplayName ## DOE Jane
                    Write-Verbose $Content.field05 ## Corporate
                    Write-Verbose $Content.field06 ## Direction
                    Write-Verbose $Content.field07 ## Chief executive officer
                    Write-Verbose $Content.field08 ## 110
                    Write-Verbose $Content.field09 ## +33 1 60 84 00 26
                    Write-Verbose $Content.field10 ## 75008
                    Write-Verbose $Content.field11 ## Paris
                    Write-Verbose $Content.field12 ## 55 Rue du Faubourg Saint-Honoré
                    Write-Verbose $Content.field13 ## Corporate User
                    Write-Verbose $Content.field14 ## OU=Users,OU=Paris,OU=Sites,OU=CORP,DC=corp,DC=priv
                    Write-Verbose $Content.field15 ## FR
                    Write-Verbose $DepartmentGroup ## GRP_DPT_DIR
                    Write-Verbose $BusinessGroup ## GRP_MET_DIR


                    $ADUserExist = $(try {Get-ADUser $SamAccountName} catch {$null})
                    If ($ADUserExist)
                    {
                        Write-Host "The user $Name already exists"
                    }
                    else
                    {
                        New-ADuser -Name $Name `
                        -GivenName $GivenName `
                        -Surname $SurName `
                        -Path $Content.field04 `
                        -AccountPassword $Password `
                        -EmailAddress $Email `
                        -SamAccountName $SamAccountName `
                        -UserPrincipalName $UserPrincipalName `
                        -DisplayName $DisplayName `
                        -Company $Content.field05 `
                        -Department $Content.field06 `
                        -Title $Content.field07 `
                        -Office $Content.field08 `
                        -OfficePhone $Content.field09 `
                        -PostalCode $Content.field10 `
                        -City $Content.field11 `
                        -StreetAddress $Content.field12 `
                        -Description $Content.field13 `
                        -Country $Content.field15 `
                        -Initials $Initials `
                        -Enabled $true `
                        -CannotChangePassword $true `
                        -PasswordNeverExpires $true

                        If ($Content.field14)
                        {
                            Set-ADUser -Identity $SamAccountName `
                            -Manager $Content.field14
                        }

                        ## If the Departement Group exist
                        $ADGroupExistDep = $(try {Get-ADGroup $DepartmentGroup} catch {$null})
                        If ($ADGroupExistDep)
                        {
                            Add-ADGroupMember $DepartmentGroup -Members $SamAccountName
                        }

                        ## If the Business Group exist
                        $ADGroupExistBus = $(try {Get-ADGroup $BusinessGroup} catch {$null})
                        If ($ADGroupExistBus)
                        {
                            Add-ADGroupMember $BusinessGroup -Members $SamAccountName
                        }

                        Write-Host "The user $Name was created"
                    }
                ; Break }
                ## End create Users ##

                ## Start create Computers ##
                'Computers' {
                    ## Name ##
                    $Name = $Content.field02

                    ## SamAccountName ##
                    $SamAccountName = $Content.field02

                    ## DNSHostName ##
                    $DNSHostName = $Name+""+$Domain

                    Write-Verbose $Name ## WD01
                    Write-Verbose $SamAccountName ## WD01
                    Write-Verbose $DNSHostName
                    Write-Verbose $Content.field03 ## OU=Computers,OU=Paris,OU=Sites,OU=CORP,DC=corp,DC=priv
                    Write-Verbose $OperatingSystem ## Windows 10 Enterprise
                    Write-Verbose $OperatingSystemVersion ## 10.0 (17763)

                    ## If the Computer exist
                    $ADComputerExist = $(try {Get-ADComputer $Name} catch {$null})
                    If ($ADComputerExist)
                    {
                        Write-Host "The Computer $Name already exists"
                    }
                    else
                    {
                        New-ADComputer -Name $Name `
                        -SamAccountName $SamAccountName  `
                        -DNSHostName $DNSHostName `
                        -Path $Content.field03 `
                        -OperatingSystem $OperatingSystem `
                        -OperatingSystemVersion $OperatingSystemVersion

                        Write-Host "The Computer $Name was created"
                    }
                ; Break }
                ## End create Computers ##

                # Start default switch #
                Default {
                     Write-Host "The file format is invalid"
                } # End default switch #
            } # End of switch #
        } # End of foreach #
    }
    else
    {
        Write-Error "Could not find the file $CSVFile"
    }
} ## End to create Fake Company ##
```

<p style="text-align: justify;"><strong>GitHub :</strong> <a href="https://github.com/vlepineadm/FakeCompany">Here is the link of the repository.</a></p>