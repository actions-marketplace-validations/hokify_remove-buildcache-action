We are using github actions to run our tests, all works fine, except it takes ages to compile everything. Therefore we were looking into caching artifacts and quickly implemented actions/cache (y).
But then it turned out, even though files are pre-compiled, nothing has changed and the buildcache info file is valid, typecript still recompiles everything.
As it turned out, github checkout (even with fetch-detph: 0) is not restoring last modified time of a file. Therefore typescript always assumes that the buildcache is invalid, even if you just compiled in in a job before that.

Therefore I like to write down how we have solved this:
1.) you need to ensure last modified time is the same
2.) ensure that .tsbuildinfo and build artifacts are included in the cache
3.) ensure that changed files are rebuild, even the buildinfo cache is still "valid".

What we did:
1.) detect what has changed (we need this later on)
```
- uses: dorny/paths-filter@v2
        id: filter
        with:
          list-files: 'json'
          filters: |
            packages: // <-- specify all your monorepo locations (usually you probabaly just have one)
              - 'packages/**'
            infrastructure:
              - 'infrastructure/**'
            services:
              - 'services/**'
```
2.) restore modified time of files via git-restore-mtime
```
      - name: Restore mtime for git checkout
        run: sudo apt install git-restore-mtime && sudo curl -o /usr/lib/git-core/git-restore-mtime https://raw.githubusercontent.com/MestreLion/git-tools/v2020.09/git-restore-mtime && git restore-mtime

```

3.) set up cache (include node_modules, build artifacts and tsbuildinfo!)
```
- name: Setup cache for build and dependencies
        uses: actions/cache@v2
        id: cache
        with:
          path: |
            node_modules
            */*/node_modules
            */*/dist
            */*/.tsbuildinfo
            ~/.npm
          key: ....
```

4.)  ensure buildcache is always in a valid state, therefore remove tsbuildinfo files from directories that have changed.
  
```
  - name: Remove buildcache infos
        uses: hokify/remove-buildcache-action@v1
        with:
          changed-files: ${{ steps.filter.outputs.packages_files }}
          tsBuildInfoFile: '.tsbuildinfo'
```

Hope this helps, it's just a very plain description, but I was looking for this for ages, and couldn't find anything.
