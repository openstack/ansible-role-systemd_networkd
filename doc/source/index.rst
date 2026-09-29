===========================================
systemd_networkd role for OpenStack-Ansible
===========================================

.. toctree::
   :maxdepth: 2

   user-guide.rst

This role will configure Systemd units:

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.

Example playbook
~~~~~~~~~~~~~~~~

To get role requirements you can use use the ``ansible-galaxy`` command on the
``requirements.yml`` file. You need to install requirements **before**
running this role.

..code-block:: bash

  # ansible-galaxy install -r requirements.yml

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports one tag: ``systemd-init``.
