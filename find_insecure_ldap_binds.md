# Finding Insecure LDAP Binds



__ADDC__
```powershell
$Events = Find-Events -Report LdapBindingsDetails -DatesRange PastHour -DetectDC

$Events.LdapBindingsDetails | export-csv ldapBindsReport.csv -NoTypeInformation
```