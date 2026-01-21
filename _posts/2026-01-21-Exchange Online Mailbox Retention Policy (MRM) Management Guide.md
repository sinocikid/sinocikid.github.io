# Exchange Online Mailbox Retention Policy (MRM) Management Guide

## 1. Check Mailbox Retention Policy Configuration

### View current retention policy assigned to mailbox
```powershell
Get-Mailbox "user" | Select RetentionPolicy
```

### Check if MRM processing is enabled
```powershell
Get-Mailbox "user" | Select ElcProcessingDisabled
```
- Returns `False` = Enabled (normal)
- Returns `True` = Disabled (needs to be enabled)

### Check mailbox archive status
```powershell
Get-Mailbox "user" | Select ArchiveStatus,ArchiveName
```
- `ArchiveStatus: Active` = Archive enabled
- `ArchiveStatus: None` = Archive needs to be enabled

---

## 2. Configure Retention Policy and Archive

### Assign retention policy to mailbox
```powershell
Set-Mailbox "user" -RetentionPolicy "PolicyName"
```

### Enable mailbox archive
```powershell
Enable-Mailbox "user" -Archive
```

---

## 3. View Retention Policy Details

### View retention tags linked to policy
```powershell
Get-RetentionPolicy "PolicyName" | Select Name,RetentionPolicyTagLinks
```

### View detailed retention tag rules (Important)
```powershell
Get-RetentionPolicy "PolicyName" | Select -ExpandProperty RetentionPolicyTagLinks | ForEach-Object {Get-RetentionPolicyTag $_ | Select Name,Type,AgeLimitForRetention,RetentionAction,RetentionEnabled}
```

**Key Field Explanations:**
- `AgeLimitForRetention`: Retention period (e.g., `4015.00:00:00` = 4015 days ≈ 11 years)
- `RetentionAction`: Action type (`MoveToArchive` = Move to archive, `DeleteAndAllowRecovery` = Delete)
- `Type`: Scope (`All` = All items)

---

## 4. Manually Trigger MRM Processing (Very Important)

### Immediately trigger Managed Folder Assistant
```powershell
Start-ManagedFolderAssistant "user"
```

**Notes:**
- Exchange Online automatically runs MRM every 7 days by default
- This command triggers processing immediately without waiting
- Wait 15-60 minutes after execution to see results

---

## 5. Verify MRM Running Status

### Check MRM last run time (may not be accurate)
```powershell
Get-MailboxStatistics "user" | Select DisplayName,ElcLastRunTime,ElcLastSuccessTimeStamp
```

**Note:** The `ElcLastRunTime` field may not update in Exchange Online even when MRM is running normally

### Check archive mailbox content (Most reliable verification method) ✅
```powershell
Get-MailboxStatistics "user" -Archive | Select ItemCount,TotalItemSize
```

**This is the best way to verify MRM is working:**
- Run this command periodically
- If `ItemCount` and `TotalItemSize` increase, items are being archived
- More reliable than `ElcLastRunTime`

### Check primary mailbox statistics
```powershell
Get-MailboxStatistics "user" | Select ItemCount,TotalItemSize,LastLogonTime
```

---

## Troubleshooting Workflow

### Issue: MRM appears not to be running

1. Check if retention policy is assigned
2. Verify `ElcProcessingDisabled` is `False`
3. Check if archive mailbox is enabled
4. Manually trigger `Start-ManagedFolderAssistant`
5. Wait 30-60 minutes and check archive mailbox item count

### Issue: `ElcLastRunTime` remains empty

- **This is normal behavior** and doesn't mean MRM isn't running
- Use `Get-MailboxStatistics -Archive` to verify by checking archive growth

### Issue: No items being archived

- Check the retention policy's `AgeLimitForRetention` (retention days)
- Confirm primary mailbox contains items older than the age limit
- Use `Get-MailboxFolderStatistics -IncludeOldestAndNewestItems` to view oldest item dates

---

## Best Practices

1. **Monitor archive growth regularly** instead of relying on `ElcLastRunTime`
2. **Manually trigger immediately after assigning new policy** using `Start-ManagedFolderAssistant`
3. **Regularly check archive mailbox capacity** to avoid quota exceeded
4. **Document retention policy rules** to ensure compliance requirements are met