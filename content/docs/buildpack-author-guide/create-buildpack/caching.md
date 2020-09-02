+++
title="Improving performance with caching"
weight=407
+++

<!-- test:suite=create-buildpack;weight=7 -->

We can improve performance by caching dependencies between builds, only re-downloading when necessary. To begin, let's create a cacheable `bundler` layer.

## Creating the `bundler` layer

To do this, replace the following lines in the `build` script:

```bash
echo "---> Installing gems"
bundle install
```

with the following:

```bash
echo "---> Installing gems"
bundlerlayer="$layersdir/bundler"
mkdir -p "$bundlerlayer"
echo -e 'cache = true\nlaunch = true' > "$bundlerlayer.toml"
bundle install --path "$bundlerlayer" --binstubs "$bundlerlayer/bin"
```

Your full `ruby-buildpack/bin/build` script should now look like the following:

<!-- test:file=ruby-buildpack/bin/build -->
```bash
#!/usr/bin/env bash
set -eo pipefail

echo "---> Ruby Buildpack"

# 1. GET ARGS
layersdir=$1

# 2. DOWNLOAD RUBY
echo "---> Downloading and extracting Ruby"
rubylayer="$layersdir"/ruby
mkdir -p "$rubylayer"
ruby_url=https://s3-external-1.amazonaws.com/heroku-buildpack-ruby/heroku-18/ruby-2.5.1.tgz
wget -q -O - "$ruby_url" | tar -xzf - -C "$rubylayer"

# 3. MAKE RUBY AVAILABLE DURING LAUNCH
echo -e 'launch = true' > "$rubylayer.toml"

# 4. MAKE RUBY AVAILABLE TO THIS SCRIPT
export PATH="$rubylayer"/bin:$PATH
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH:+${LD_LIBRARY_PATH}:}"$rubylayer/lib"

# 5. INSTALL BUNDLER
echo "---> Installing bundler"
gem install bundler --no-ri --no-rdoc

# ======= MODIFIED =======
# 6. INSTALL GEMS
echo "---> Installing gems"
bundlerlayer="$layersdir/bundler"
mkdir -p "$bundlerlayer"
echo -e 'cache = true\nlaunch = true' > "$bundlerlayer.toml"
bundle install --path "$bundlerlayer" --binstubs "$bundlerlayer/bin"

# 7. SET DEFAULT START COMMAND
cat > "$layersdir/launch.toml" <<EOL
# our web process
[[processes]]
type = "web"
command = "bundle exec ruby app.rb"

# our worker process
[[processes]]
type = "worker"
command = "bundle exec ruby worker.rb"
EOL
```

Now when we run:

<!-- test:exec -->
```bash
pack build test-ruby-app --path ./ruby-sample-app --buildpack ./ruby-buildpack
```

You will see something similar to the following during the `EXPORTING` phase:

<!-- test:assert=contains -->
```text
[exporter] Adding layer 'com.examples.buildpacks.ruby:bundler'
```

## Caching dependencies

Now, let's implement the caching logic. We'll first need to create a `ruby-sample-app/Gemfile.lock` file with the contents given below:

> Typically you would run `bundle install` locally to generate this file, but for the sake 
> of simplicity we'll create `ruby-sample-app/Gemfile.lock` manually.

<!-- test:file=ruby-sample-app/Gemfile.lock -->
```text
GEM
  remote: https://rubygems.org/
  specs:
    mustermann (1.0.3)
    rack (2.0.7)
    rack-protection (2.0.7)
      rack
    sinatra (2.0.7)
      mustermann (~> 1.0)
      rack (~> 2.0)
      rack-protection (= 2.0.7)
      tilt (~> 2.0)
    tilt (2.0.9)

PLATFORMS
  ruby

DEPENDENCIES
  sinatra

BUNDLED WITH
   2.0.2
```

Replace the gem installation logic from the previous step:

```bash
# ...

echo "---> Installing gems"
bundlerlayer="$layersdir/bundler"
mkdir -p "$bundlerlayer"
echo -e 'cache = true\nlaunch = true' > "$bundlerlayer.toml"
bundle install --path "$bundlerlayer" --binstubs "$bundlerlayer/bin"

# ...
```

with the new logic below that checks to see if any gems have been changed. This simply creates a checksum for the previous `Gemfile.lock` and compares it to the checksum of the current `Gemfile.lock`. If they are the same, the gems are reused. If they are not, the new gems are installed.

We'll now write additional metadata to our `bundler.toml` of the form `cache = true` and `launch = true`. This directs the lifecycle to cache our gems and provide them when launching our application. With `cache = true` the lifecycle can keep existing gems around so that build times are fast, even with minor `Gemfile.lock` changes.

```bash
# Compares previous Gemfile.lock checksum to the current Gemfile.lock
bundlerlayer="$layersdir/bundler"
local_bundler_checksum=$(sha256sum Gemfile.lock | cut -d ' ' -f 1) 
remote_bundler_checksum=$(cat "$bundlerlayer.toml" | yj -t | jq -r .metadata 2>/dev/null || echo 'not found')

if [[ -f Gemfile.lock && $local_bundler_checksum == $remote_bundler_checksum ]] ; then
    # Determine if no gem dependencies have changed, so it can reuse existing gems without running bundle install
    echo "---> Reusing gems"
    bundle config --local path "$bundlerlayer" >/dev/null
    bundle config --local bin "$bundlerlayer/bin" >/dev/null
else
    # Determine if there has been a gem dependency change and install new gems to the bundler layer; re-using existing and un-changed gems
    echo "---> Installing gems"
    mkdir -p "$bundlerlayer"
    echo -e "cache = true\nlaunch = true\nmetadata = \"$local_bundler_checksum\"" > "$bundlerlayer.toml"
    bundle install --path "$bundlerlayer" --binstubs "$bundlerlayer/bin"
fi
```

Your full `ruby-buildpack/bin/build` script will now look like this:

<!-- test:file=ruby-buildpack/bin/build -->
```bash
#!/usr/bin/env bash
set -eo pipefail

echo "---> Ruby Buildpack"

# 1. GET ARGS
layersdir=$1

# 2. DOWNLOAD RUBY
echo "---> Downloading and extracting Ruby"
rubylayer="$layersdir"/ruby
mkdir -p "$rubylayer"
ruby_url=https://s3-external-1.amazonaws.com/heroku-buildpack-ruby/heroku-18/ruby-2.5.1.tgz
wget -q -O - "$ruby_url" | tar -xzf - -C "$rubylayer"

# 3. MAKE RUBY AVAILABLE DURING LAUNCH
echo -e 'launch = true' > "$rubylayer.toml"

# 4. MAKE RUBY AVAILABLE TO THIS SCRIPT
export PATH="$rubylayer"/bin:$PATH
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH:+${LD_LIBRARY_PATH}:}"$rubylayer/lib"

# 5. INSTALL BUNDLER
echo "---> Installing bundler"
gem install bundler --no-ri --no-rdoc

# ======= MODIFIED =======
# 6. INSTALL GEMS
# Compares previous Gemfile.lock checksum to the current Gemfile.lock
bundlerlayer="$layersdir/bundler"
local_bundler_checksum=$(sha256sum Gemfile.lock | cut -d ' ' -f 1) 
remote_bundler_checksum=$(cat "$bundlerlayer.toml" | yj -t | jq -r .metadata 2>/dev/null || echo 'not found')

if [[ -f Gemfile.lock && $local_bundler_checksum == $remote_bundler_checksum ]] ; then
    # Determine if no gem dependencies have changed, so it can reuse existing gems without running bundle install
    echo "---> Reusing gems"
    bundle config --local path "$bundlerlayer" >/dev/null
    bundle config --local bin "$bundlerlayer/bin" >/dev/null
else
    # Determine if there has been a gem dependency change and install new gems to the bundler layer; re-using existing and un-changed gems
    echo "---> Installing gems"
    mkdir -p "$bundlerlayer"
    echo -e "cache = true\nlaunch = true\nmetadata = \"$local_bundler_checksum\"" > "$bundlerlayer.toml"
    bundle install --path "$bundlerlayer" --binstubs "$bundlerlayer/bin"
fi

# 7. SET DEFAULT START COMMAND
cat > "$layersdir/launch.toml" <<EOL
# our web process
[[processes]]
type = "web"
command = "bundle exec ruby app.rb"

# our worker process
[[processes]]
type = "worker"
command = "bundle exec ruby worker.rb"
EOL
```

Now when you build your app:

<!-- test:exec -->
```text
pack build test-ruby-app --path ./ruby-sample-app --buildpack ./ruby-buildpack
```

it will download the gems:

<!-- test:assert=contains -->
```text
===> BUILDING
[builder] ---> Ruby Buildpack
[builder] ---> Downloading and extracting Ruby
[builder] ---> Installing bundler
[builder] Successfully installed bundler-2.1.4
[builder] 1 gem installed
[builder] ---> Installing gems
```

If you build the app again:

<!-- test:exec -->
```bash
pack build test-ruby-app --path ./ruby-sample-app --buildpack ./ruby-buildpack
```

you will see the new caching logic at work during the `BUILDING` phase:

<!-- test:assert=contains -->
```text
===> BUILDING
[builder] ---> Ruby Buildpack
[builder] ---> Downloading and extracting Ruby
[builder] ---> Installing bundler
[builder] Successfully installed bundler-2.1.4
[builder] 1 gem installed
[builder] ---> Reusing gems
```

Next, let's see how buildpack users may be able to provide configuration to the buildpack.

---

<a href="/docs/buildpack-author-guide/create-buildpack/make-buildpack-configurable" class="button bg-pink">Next Step</a>
