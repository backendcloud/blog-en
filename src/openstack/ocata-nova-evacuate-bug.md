release time :2017-06-28 18:24

# Phenomenon

Execute the nova evacuate operation, but there is a problem during the rebuild. After a certain step, the error "rebuild virtual machine has been deleted" is reported.

# reason

When an instance is evacuated from one node to another, an InstanceNotFound exception is expected to occur during the execution of the check_instance_exists method during the rebuild process.

The check_instance_exists method is used to ensure that the instance does not exist on the new node, and if it exists, an exception will be thrown.

The check_instance_exists in the actual code reported this exception when it should not report the InstanceNotFound exception, and the exception of the method was not added to capture the InstanceNotFound exception.

# community fixes

nova/virt/libvirt/driver.py

```python
def instance_exists(self, instance):
    """Efficient override of base instance_exists method."""
    try:
        self._host.get_guest(instance)
        return True
    except exception.InternalError:
        return False
```

change to:

```python
def instance_exists(self, instance):
    """Efficient override of base instance_exists method."""
    try:
        self._host.get_guest(instance)
        return True
    except (exception.InternalError, exception.InstanceNotFound):
        return False
```



# unit test

```python
@mock.patch.object(host.Host, 'get_guest')
def test_instance_exists(self, mock_get_guest):
    drvr = libvirt_driver.LibvirtDriver(fake.FakeVirtAPI(), False)
    self.assertTrue(drvr.instance_exists(None))

    mock_get_guest.side_effect = exception.InstanceNotFound
    self.assertFalse(drvr.instance_exists(None))

    mock_get_guest.side_effect = exception.InternalError
    self.assertFalse(drvr.instance_exists(None))
```
