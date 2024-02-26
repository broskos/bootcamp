# Todo: 
Followings were changed: 
```
pre-reqs --> secrets
site-configs --> clusters
clusters/resources --> policies/manifests
site-policies --> policies 
hub-1 --> site-group-1
```

## Cloning Git Repo: 

```
mkdir -p ~/5g-deployment-lab/
git clone http://student:student@infra.5g-deployment.lab:3000/student/ztp-repository.git ~/5g-deployment-lab/ztp-repository/
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

## Create Git Structure: 

```
cd ~/5g-deployment-lab/ztp-repository/
mkdir -p clusters/{site-group-1,site-group-1/secrets/sno2,site-group-1/secrets/sno3,site-group-1/sno2-extra-manifest,site-group-1/sno3-extra-manifest}
mkdir -p policies/{site-specific-policies,resources,configuration-version-2024-03-04,configuration-version-2024-03-04/source-crs,configuration-version-2024-03-04/manifests}
touch clusters/{site-group-1,site-group-1/secrets/sno2,site-group-1/secrets/sno3}/.gitkeep
touch policies/{site-specific-policies,resources,configuration-version-2024-03-04,configuration-version-2024-03-04/source-crs,configuration-version-2024-03-04/manifests}/.gitkeep
git add --all
git commit -m 'Initialized repo structure'
git push origin main
```
