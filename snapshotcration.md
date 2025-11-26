graph TD
    Start([Start: User Launches<br/>AAP Job Template]) --> Input[Collect Survey Inputs:<br/>- instance_ids CSV<br/>- aws_access_key<br/>- aws_secret_key<br/>- aws_session_token<br/>- aws_region<br/>- reference_number<br/>- optional_email_id]
    
    Input --> ValidateInput{Validate<br/>Mandatory Inputs}
    ValidateInput -->|Missing Fields| FailInput[Fail Job:<br/>Missing Required Parameters]
    ValidateInput -->|Valid| ParseCSV[Parse CSV Instance IDs<br/>to List]
    
    ParseCSV --> ValidateFormat{Validate<br/>Instance ID Format<br/>i-[a-f0-9]8,17}
    ValidateFormat -->|Invalid Format| FailFormat[Fail Job:<br/>Invalid Instance ID Format]
    ValidateFormat -->|Valid| BuildEmail[Build Email<br/>Recipients List]
    
    BuildEmail --> TestAWS[Test AWS Credentials<br/>ec2_instance_info API Call]
    TestAWS -->|Auth Failed| FailAuth[Fail Job:<br/>AWS Authentication Failed]
    TestAWS -->|Auth Success| DiscoverInstances[Gather EC2 Instance<br/>Information for All IDs]
    
    DiscoverInstances --> CheckInstances{Instances<br/>Found?}
    CheckInstances -->|No Instances| FailNoInstances[Fail Job:<br/>No Instances Found]
    CheckInstances -->|Instances Found| CheckMissing{Any Missing<br/>Instance IDs?}
    
    CheckMissing -->|Yes| WarnMissing[Log Warning:<br/>Display Missing Instance IDs]
    CheckMissing -->|No| ExtractVolumes[Extract EBS Volumes<br/>from All Instances]
    WarnMissing --> ExtractVolumes
    
    ExtractVolumes --> BuildVolumeList[Build discovered_volumes List:<br/>- instance_id<br/>- instance_name<br/>- instance_tags<br/>- volume_id<br/>- device_name]
    
    BuildVolumeList --> LogDiscovery[Log Discovery Summary:<br/>Total Volumes Found<br/>Total Size]
    
    LogDiscovery --> GenInitialEmail[Generate Initial<br/>Assessment Email:<br/>HTML Table with<br/>Discovered Volumes]
    
    GenInitialEmail --> SendInitialEmail[Send Initial Email:<br/>Subject: Initial Assessment]
    SendInitialEmail -->|Email Failed| WarnEmail1[Warn: Email Failed<br/>Continue Anyway]
    SendInitialEmail -->|Email Sent| CreateSnapshots
    WarnEmail1 --> CreateSnapshots
    
    CreateSnapshots[Create Snapshots Loop:<br/>For Each Volume]
    CreateSnapshots --> SetSnapshotDetails[Set Snapshot Details:<br/>- Description: ref_inst_vol<br/>- Replicate Instance Tags<br/>- Add VolumeId Tag<br/>- Add InstanceId Tag<br/>- Add ReferenceNumber Tag<br/>- Add CreatedBy Tag]
    
    SetSnapshotDetails --> CallSnapshotAPI[Call ec2_snapshot:<br/>state: present<br/>wait: false]
    
    CallSnapshotAPI -->|Snapshot Failed| FailSnapshot[Fail Job:<br/>Critical Snapshot Error]
    CallSnapshotAPI -->|Snapshot Initiated| BuildResults[Build snapshot_results List:<br/>- instance_id<br/>- volume_id<br/>- snapshot_id<br/>- status: initiated]
    
    BuildResults --> CheckMoreVolumes{More Volumes<br/>to Process?}
    CheckMoreVolumes -->|Yes| CreateSnapshots
    CheckMoreVolumes -->|No| LogInitiated[Log: All Snapshots<br/>Initiated Successfully]
    
    LogInitiated --> WaitLoop[Wait for Completion Loop:<br/>For Each Snapshot]
    
    WaitLoop --> QueryStatus[Query Snapshot Status:<br/>ec2_snapshot_info]
    
    QueryStatus --> CheckComplete{Snapshot<br/>State =<br/>completed?}
    
    CheckComplete -->|No| CheckRetry{Retries<br/>< 60?}
    CheckRetry -->|Yes| Wait30[Wait 30 Seconds]
    Wait30 --> QueryStatus
    CheckRetry -->|No, Timeout| MarkFailed[Mark Snapshot<br/>Status: failed]
    
    CheckComplete -->|Yes| MarkCompleted[Mark Snapshot<br/>Status: completed<br/>Progress: 100%]
    
    MarkCompleted --> CheckMoreSnapshots{More Snapshots<br/>to Verify?}
    MarkFailed --> CheckMoreSnapshots
    
    CheckMoreSnapshots -->|Yes| WaitLoop
    CheckMoreSnapshots -->|No| CalcStats[Calculate Statistics:<br/>- total_snapshots<br/>- successful_snapshots<br/>- failed_snapshots]
    
    CalcStats --> GenFinalEmail[Generate Final<br/>Assessment Email:<br/>HTML Table with Results]
    
    GenFinalEmail --> SendFinalEmail[Send Final Email:<br/>Subject: Final Assessment<br/>Success: X/Y]
    
    SendFinalEmail -->|Email Failed| WarnEmail2[Warn: Email Failed<br/>Data in Job Logs]
    SendFinalEmail -->|Email Sent| DisplaySummary
    WarnEmail2 --> DisplaySummary
    
    DisplaySummary[Display Job<br/>Completion Summary:<br/>- Instances Processed<br/>- Volumes Processed<br/>- Snapshots Created<br/>- Success Rate]
    
    DisplaySummary --> CheckSuccess{All Snapshots<br/>Successful?}
    
    CheckSuccess -->|Yes| Success([End: Job SUCCESS])
    CheckSuccess -->|No| PartialSuccess([End: Job PARTIAL SUCCESS])
    
    FailInput --> End([End: Job FAILED])
    FailFormat --> End
    FailAuth --> End
    FailNoInstances --> End
    FailSnapshot --> End
    
    style Start fill:#d4edda,stroke:#28a745,stroke-width:3px
    style Success fill:#d4edda,stroke:#28a745,stroke-width:3px
    style PartialSuccess fill:#fff3cd,stroke:#ffc107,stroke-width:3px
    style End fill:#f8d7da,stroke:#dc3545,stroke-width:3px
    
    style ValidateInput fill:#e7f3ff,stroke:#0066cc
    style TestAWS fill:#e7f3ff,stroke:#0066cc
    style CheckInstances fill:#e7f3ff,stroke:#0066cc
    style CheckComplete fill:#e7f3ff,stroke:#0066cc
    style CheckSuccess fill:#e7f3ff,stroke:#0066cc
    
    style FailInput fill:#f8d7da,stroke:#dc3545
    style FailFormat fill:#f8d7da,stroke:#dc3545
    style FailAuth fill:#f8d7da,stroke:#dc3545
    style FailNoInstances fill:#f8d7da,stroke:#dc3545
    style FailSnapshot fill:#f8d7da,stroke:#dc3545
    
    style SendInitialEmail fill:#fff3cd,stroke:#ffc107
    style SendFinalEmail fill:#fff3cd,stroke:#ffc107
    style CreateSnapshots fill:#d1ecf1,stroke:#0c5460
    style WaitLoop fill:#d1ecf1,stroke:#0c5460