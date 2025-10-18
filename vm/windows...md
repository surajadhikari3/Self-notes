
cd source/vm-practise/boca-vagrant/windows

|Task|Command|Purpose|
|---|---|---|
|Suspend VM (quick save)|`vagrant suspend`|Pause and resume fast|
|Full shutdown|`vagrant halt`|Clean stop|
|Boot VM|`vagrant up`|Start or resume|
|Reprovision (if needed)|`vagrant reload --provision`|Apply config changes|
|Status check|`vagrant status`|Verify running / suspended|
![[Pasted image 20251018171548.png]]Make sure to add the bridged network in the network adapter to bypass the mac config and to run the desired sites...