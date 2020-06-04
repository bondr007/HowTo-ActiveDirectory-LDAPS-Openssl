# Finding Insecure LDAP Binds



__ADDC__
```powershell
#Last3days PastHour
$Events = Find-Events -Report LdapBindingsDetails -DatesRange Last3days -DetectDC

$Events.LdapBindingsDetails | export-csv ldapBindsReport.csv -NoTypeInformation
```