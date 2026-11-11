This file will explain all decisions made during the project

.gitignore : 
  - Why *.tfstate matters most - These files contain plaintext secrets and my full infrastructure blueprint. If tihs got committed to a publlic repo, that's a common way people leak credentials. Since we're using remote S3 backend(not local state), I most likely wont even generate .tfstate files that much, but the rule stays the same as permanent safeguard either way.
  - Why .terraform.lock.hcl is the only exception (! unignores it) - This file provides the exact provider version/checksum that was downloaded, so everyone running this code gets the identical provider build. This one SHOULD be committed - its how teams keep reproducible builds.

workflows - terraform-validate.yml :
  - Why -backend=false on init - the S3 remote backend hasn't been built yet (that’s Milestone 1, #4-6, next). This flag lets validate check syntax without needing a real backend to connect to.
  - @v4 and @v3 are mutable tags, meaning the maintainer can repoint them to a different commit at any time. If action/checkout's repo were ever compromised, an attacker could push mailcious code and retag v4 to point at it. The workflow would silently pull the compromised version on the next run, with zero code change on my side # example of a possible supply chain attacker vector