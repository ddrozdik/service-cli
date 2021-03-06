# Build mhsendmail
FROM golang:1.8.3 as mhbuild
ENV MHSENDMAIL_VERSION=0.2.0
RUN set -xe; \
  go get -d github.com/mailhog/mhsendmail; \
  cd /go/src/github.com/mailhog/mhsendmail; \
  git checkout tags/v${MHSENDMAIL_VERSION}; \
  go get github.com/mailhog/mhsendmail

# Build the image
FROM php:5.6-fpm

ARG DEBIAN_FRONTEND=noninteractive

# Prevent services autoload (http://jpetazzo.github.io/2013/10/06/policy-rc-d-do-not-start-services-automatically/)
RUN set -xe; \
	echo '#!/bin/sh\nexit 101' > /usr/sbin/policy-rc.d && chmod +x /usr/sbin/policy-rc.d

# Install basic packages
RUN set -xe; \
	apt-get update >/dev/null; apt-get -y --force-yes --no-install-recommends install >/dev/null \
		apt-transport-https \
		ca-certificates \
		curl \
		gnupg \
		locales \
		wget \
	;\
	# Cleanup
	apt-get clean; rm -rf /var/lib/apt/lists/*

# Set timezone and locale
RUN set -xe; \
	dpkg-reconfigure locales; \
	locale-gen C.UTF-8; \
	/usr/sbin/update-locale LANG=C.UTF-8
ENV LC_ALL C.UTF-8

# Enable additional repos
RUN set -xe; \
	sed -i 's/main/main contrib non-free/' /etc/apt/sources.list; \
	# backports repo
	echo "deb http://ftp.debian.org/debian jessie-backports main" | tee /etc/apt/sources.list.d/backports.list; \
	# blackfire.io repo
	curl -sSL https://packagecloud.io/gpg.key | apt-key add -; \
	echo "deb https://packages.blackfire.io/debian any main" | tee /etc/apt/sources.list.d/blackfire.list; \
	# git-lfs repo
	curl -sSL https://packagecloud.io/github/git-lfs/gpgkey | apt-key add -; \
	echo 'deb https://packagecloud.io/github/git-lfs/debian/ jessie main' | tee /etc/apt/sources.list.d/github_git-lfs.list; \
	echo 'deb-src https://packagecloud.io/github/git-lfs/debian/ jessie main' | tee -a /etc/apt/sources.list.d/github_git-lfs.list;

# Additional packages
RUN set -xe; \
	# Create man direcotries, otherwise some packages may not install (e.g. postgresql-client)
	# This should be a temporary workaround until fixed upstream: https://github.com/debuerreotype/debuerreotype/issues/10
	mkdir -p /usr/share/man/man1 /usr/share/man/man7; \
	apt-get update >/dev/null; apt-get -y --force-yes --no-install-recommends install >/dev/null \
		cron \
		dnsutils \
		git-lfs \
		ghostscript \
		# html2text binary - used for self-testing (php-fpm)
		html2text \
		imagemagick \
		less \
		# cgi-fcgi binary - used for self-testing (php-fpm)
		libfcgi-bin \
		mc \
		mysql-client \
		nano \
		openssh-client \
		openssh-server \
		postgresql-client \
		procps \
		pv \
		rsync \
		sudo \
		supervisor \
		unzip \
		zip \
		zsh \
	;\
	# More recent version of git to get composer's git cache.
	apt-get -y --force-yes --no-install-recommends -t jessie-backports install git >/dev/null ;\
	# Cleanup
	apt-get clean; rm -rf /var/lib/apt/lists/*

RUN set -xe; \
	# Create a regular user/group "docker" (uid = 1000, gid = 1000 ) with access to sudo
	groupadd docker -g 1000; \
	useradd -m -s /bin/bash -u 1000 -g 1000 -G sudo -p docker docker; \
	echo 'docker ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# Install gosu and give access to the docker user primary group to use it.
# gosu is used instead of sudo to start the main container process (pid 1) in a docker friendly way.
# https://github.com/tianon/gosu
RUN set -xe; \
	curl -sSL "https://github.com/tianon/gosu/releases/download/1.10/gosu-$(dpkg --print-architecture)" -o /usr/local/bin/gosu; \
	chown root:"$(id -gn docker)" /usr/local/bin/gosu; \
	chmod +sx /usr/local/bin/gosu

# Configure sshd (for use PHPStorm's remote interpreters and tools integrations)
# http://docs.docker.com/examples/running_ssh_service/
RUN set -xe; \
	mkdir /var/run/sshd; \
	echo 'docker:docker' | chpasswd; \
	sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config; \
	# SSH login fix. Otherwise user is kicked off after login
	sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd; \
	echo "export VISIBLE=now" >> /etc/profile
ENV NOTVISIBLE "in users profile"

# PHP
RUN set -xe; \
	buildDeps=" \
		freetds-dev \
		g++ \
		libc-client2007e-dev \
		libfreetype6-dev \
		libgpgme11-dev \
		libicu-dev \
		libjpeg62-turbo-dev \
		libkrb5-dev \
		libldap2-dev \
		libmagickcore-dev \
		libmagickwand-dev \
		libmcrypt-dev \
		libmemcached-dev \
		libmhash-dev \
		libpng-dev \
		libpq-dev \
		libssh2-1-dev \
		libxslt1-dev \
		zlib1g-dev \
	"; \
	apt-get update >/dev/null; apt-get -y --force-yes --no-install-recommends install >/dev/null \
		$buildDeps \
		blackfire-php \
		libc-client2007e \
		libfreetype6 \
		libgpgme11 \
		libicu57 \
		libjpeg62-turbo \
		libldap-2.4-2 \
		libmagickcore-6.q16-3 \
		libmagickwand-6.q16-3 \
		libmcrypt4 \
		libmhash2 \
		libpng16-16 \
		libpq5 \
		libssh2-1 \
		libsybdb5 \
		libxslt1.1 \
		zlib1g \
	;\
	docker-php-ext-configure >/dev/null gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/; \
	docker-php-ext-configure >/dev/null imap --with-kerberos --with-imap-ssl; \
	docker-php-ext-configure >/dev/null ldap --with-libdir=lib/x86_64-linux-gnu/; \
	# mssql can be install in PHP 5.6 only. pdo_dblib is available in PHP 7.0+ as well.
	docker-php-ext-configure >/dev/null mssql --with-libdir=/lib/x86_64-linux-gnu/; \
	docker-php-ext-configure >/dev/null pdo_dblib --with-libdir=/lib/x86_64-linux-gnu/; \
	docker-php-ext-configure >/dev/null pgsql --with-pgsql=/usr/local/pgsql/; \
	\
	docker-php-ext-install >/dev/null -j$(nproc) \
		bcmath \
		bz2 \
		calendar\
		exif \
		gd \
		gettext \
		imap \
		intl \
		ldap \
		mcrypt \
		mssql \
		mysqli \
		opcache \
		pcntl \
		pdo_dblib \
		pdo_mysql \
		pdo_pgsql \
		pgsql \
		soap \
		sockets \
		xsl \
		zip \
	;\
	pecl update-channels; \
	pecl install >/dev/null </dev/null \
		gnupg \
		imagick \
		memcache \
		redis \
		ssh2 \
		# xdebug-2.5.5 is the last version with php5 support
		xdebug-2.5.5 \
	;\
	docker-php-ext-enable \
		gnupg \
		imagick \
		memcache \
		redis \
		ssh2 \
	;\
	# Disable xdebug by default
	rm -f /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini; \
	# Cleanup
	docker-php-source delete; \
	rm -rf /tmp/pear ~/.pearrc; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $buildDeps >/dev/null; \
	apt-get clean; rm -rf /var/lib/apt/lists/*

ENV COMPOSER_VERSION=1.6.3 \
	DRUSH_VERSION=8.1.16 \
	DRUSH_LAUNCHER_VERSION=0.6.0 \
	DRUPAL_CONSOLE_VERSION=1.7.0 \
	WPCLI_VERSION=1.5.0 \
	MG_CODEGEN_VERSION=1.10 \
	BLACKFIRE_VERSION=1.15.0 \
	GOMPLATE_VERSION=2.4.0
RUN set -xe; \
	# Composer
	curl -sSL "https://github.com/composer/composer/releases/download/${COMPOSER_VERSION}/composer.phar" -o /usr/local/bin/composer; \
	# Drush 8 (default)
	curl -sSL "https://github.com/drush-ops/drush/releases/download/${DRUSH_VERSION}/drush.phar" -o /usr/local/bin/drush8; \
	# Drush Launcher
	curl -sSL "https://github.com/drush-ops/drush-launcher/releases/download/${DRUSH_LAUNCHER_VERSION}/drush.phar" -o /usr/local/bin/drush; \
	# Drupal Console
	curl -sSL "https://github.com/hechoendrupal/drupal-console-launcher/releases/download/${DRUPAL_CONSOLE_VERSION}/drupal.phar" -o /usr/local/bin/drupal; \
	# Wordpress CLI
	curl -sSL "https://github.com/wp-cli/wp-cli/releases/download/v${WPCLI_VERSION}/wp-cli-${WPCLI_VERSION}.phar" -o /usr/local/bin/wp; \
	# Magento2 Code Generator
	curl -sSL "https://github.com/staempfli/magento2-code-generator/releases/download/${MG_CODEGEN_VERSION}/mg2-codegen.phar" -o /usr/local/bin/mg2-codegen; \
	# Blackfire CLI
	curl -sSL https://packages.blackfire.io/binaries/blackfire-agent/${BLACKFIRE_VERSION}/blackfire-cli-linux_static_amd64 -o /usr/local/bin/blackfire; \
	# gomplate
	curl -sSL https://github.com/hairyhenderson/gomplate/releases/download/v${GOMPLATE_VERSION}/gomplate_linux-amd64-slim -o /usr/local/bin/gomplate; \
	# Make all binaries executable in one shot
	cd /usr/local/bin && chmod +x composer drush8 drush drupal wp mg2-codegen blackfire gomplate;

# Node.js
ENV NODE_VERSION=8.x
RUN set -xe; \
	# Node.js repo
	curl -sSL https://deb.nodesource.com/gpgkey/nodesource.gpg.key | apt-key add -; \
	echo "deb https://deb.nodesource.com/node_${NODE_VERSION} jessie main" | sudo tee /etc/apt/sources.list.d/nodesource.list; \
	echo "deb-src https://deb.nodesource.com/node_${NODE_VERSION} jessie main" | sudo tee -a /etc/apt/sources.list.d/nodesource.list; \
	# yarn repo
	curl -sSL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -; \
	echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list; \
	apt-get update >/dev/null; apt-get -y --force-yes --no-install-recommends install >/dev/null \
		nodejs \
		yarn \
	;\
	apt-get clean; rm -rf /var/lib/apt/lists/*;

# Ruby
RUN set -xe; \
	apt-get update >/dev/null; apt-get -y --force-yes --no-install-recommends install >/dev/null \
		ruby-full \
		rlwrap \
	;\
	# Cleanup
	apt-get clean; rm -rf /var/lib/apt/lists/*; \
	# bundler
	gem install bundler >/dev/null
# Home directory for bundle installs
ENV BUNDLE_PATH .bundler

# All further RUN commands will run as the "docker" user
USER docker
ENV HOME /home/docker

# Install Composer based dependencies
ENV PATH=$PATH:$HOME/.composer/vendor/bin \
	DRUSH_LAUNCHER_FALLBACK=/usr/local/bin/drush8
ENV PATH=$PATH:$HOME/.terminus/vendor/bin \
	TERMINUS_VERSION=1.8.1
ENV PATH=$PATH:$HOME/.platformsh \
	PLATFORMSH_CLI_VERSION=3.33.5
RUN set -xe; \
	# Add composer bin directory to PATH
	echo "\n"'PATH="$PATH:$HOME/.composer/vendor/bin"' >> $HOME/.profile; \
	# Drush modules
	drush dl registry_rebuild --default-major=7 --destination=$HOME/.drush >/dev/null; \
	drush cc drush; \
	# Drupal Coder w/ a matching version of PHP_CodeSniffer
	composer global require drupal/coder >/dev/null; \
	phpcs --config-set installed_paths $HOME/.composer/vendor/drupal/coder/coder_sniffer; \
	# Composer parallel install plugin
	composer global require hirak/prestissimo >/dev/null; \
	# Terminus
	# Installed in a dedicated directory to avoid dependency conflicts.
	echo "\n"'PATH="$PATH:$HOME/.terminus/vendor/bin"' >> $HOME/.profile; \
	mkdir -p $HOME/.terminus; \
	# Run in a subshell since we are doing directory switching
	(cd $HOME/.terminus && composer require pantheon-systems/terminus:${TERMINUS_VERSION}); \
	mkdir -p $HOME/.platformsh; \
	(cd $HOME/.platformsh && curl -o platform -L "https://github.com/platformsh/platformsh-cli/releases/download/v${PLATFORMSH_CLI_VERSION}/platform.phar" && chmod 755 platform); \
	# Cleanup
	composer clear-cache

USER root

# Copy mhsendmail binary from stage 1
COPY --from=mhbuild /go/bin/mhsendmail /usr/local/bin/mhsendmail
# Copy configs and scripts
COPY --chown=docker:docker config/.acquia $HOME/.acquia
COPY --chown=docker:docker config/.docksalrc $HOME/.docksalrc
COPY --chown=docker:docker config/.drush $HOME/.drush
COPY --chown=docker:docker config/.ssh $HOME/.ssh
COPY config/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY startup.sh /opt/startup.sh
COPY healthcheck.sh /opt/healthcheck.sh
# PHP default settings, global overrides and fpm overrides
ADD https://raw.githubusercontent.com/php/php-src/PHP-${PHP_VERSION}/php.ini-development /usr/local/etc/php/php.ini
COPY config/php/zz-php.ini /usr/local/etc/php/conf.d/zz-php.ini
COPY config/php/xdebug.ini /opt/docker-php-ext-xdebug.ini
COPY config/php/zz-php-fpm.conf /usr/local/etc/php-fpm.d/zz-php-fpm.conf
# Create symlinks to project level overrides (if the source files are missing, nothing will break)
RUN set -xe; \
	ln -s /var/www/.docksal/etc/php/php.ini /usr/local/etc/php/conf.d/zzz-php.ini; \
	ln -s /var/www/.docksal/etc/php/php-fpm.conf /usr/local/etc/php-fpm.d/zzz-php-fpm.conf

ENV \
	# ssh-agent proxy socket (requires docksal/ssh-agent)
	SSH_AUTH_SOCK=/.ssh-agent/proxy-socket \
	# Set TERM so text editors/etc. can be used
	TERM=xterm \
	# Allow PROJECT_ROOT to be universally used in fin custom commands (inside and outside cli)
	PROJECT_ROOT=/var/www \
	# Default values for HOST_UID and HOST_GUI to match the default Ubuntu user. These are used in startup.sh
	HOST_UID=1000 \
	HOST_GID=1000 \
	# xdebug disabled by default
	XDEBUG_ENABLED=0

EXPOSE 9000
EXPOSE 22
EXPOSE 3000

WORKDIR /var/www

# Starter script
ENTRYPOINT ["/opt/startup.sh"]

# By default, launch supervisord to keep the container running.
CMD ["supervisord"]

# Health check script
HEALTHCHECK --interval=5s --timeout=1s --retries=12 CMD ["/opt/healthcheck.sh"]
