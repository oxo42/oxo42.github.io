---
layout: post
title:  "VMWare Workstation: Start VMs on user login"
date:   2015-10-05
categories: windows
tags: vmware
---
{% include JB/setup %}

I want to start a certain list of VMWare Workstation virtual machines on machine boot.  I created a Scheduled task that on logon runs:

{% highlight batch %}
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -NoLogo -NonInteractive -File "C:\Users\Tools\Start-VMs.ps1"
{% endhighlight %}

And the contents of `Start-VMs.ps1` are:

{% highlight powershell %}
$vmrun = 'C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe'
$vmlist = @('C:\Virtual Machines\Ubuntu\Ubuntu.vmx', 'I:\Virtual Machines\UbuntuServer\UbuntuServer.vmx')

$authd = Get-Service -Name VMAuthdService
if($authd.Status -ne 'Running') {
    $authd.Start()
}

foreach($vm in $vmlist) {
    & $vmrun start "$vm"
}
{% endhighlight %}
